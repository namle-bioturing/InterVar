# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

InterVar is a bioinformatics tool for clinical interpretation of genetic variants following the ACMG-AMP 2015 guidelines. It automatically interprets variants as "pathogenic", "likely pathogenic", "uncertain significance", "likely benign" or "benign" based on 28 evidence codes.

Key features:
- Takes VCF or ANNOVAR input formats containing genetic variants
- Integrates with ANNOVAR for variant annotation
- Evaluates 18 automated criteria (PVS1, PS1, PS4, PM1-PM5, PP2-PP3, PP5, BA1, BS1-BS2, BP1, BP3-BP4, BP6-BP7)
- Allows manual evidence input for remaining 10 criteria (PS2-PS3, PM3, PM6, PP1, PP4, BS3-BS4, BP2, BP5)
- Supports custom evidence files for fine-tuned adjustments

## Running InterVar

### Basic Commands

Run with config file (recommended for testing):
```bash
python Intervar.py -c config.ini
```

Run with VCF input:
```bash
python Intervar.py -b hg19 -i input.vcf --input_type=VCF -o output_prefix
```

Run with ANNOVAR input:
```bash
python Intervar.py -b hg19 -i input.avinput --input_type=AVinput -o output_prefix
```

Skip ANNOVAR annotation (if already done):
```bash
python Intervar.py -c config.ini --skip_annovar
```

### Example with test data:
```bash
python Intervar.py -b hg19 -i example/ex1.avinput --input_type=AVinput -o example/test_output
```

## Architecture

### Main Execution Flow

1. **Configuration Loading** (`main()` at line 1886)
   - Parses command-line options
   - Loads config.ini settings
   - Validates paths and input files

2. **Database File Loading** (`read_datasets()` at line 133)
   - Loads reference datasets from `intervardb/` directory
   - Files include LOF genes, AA changes, protein domains, OMIM data, etc.
   - All loaded into global dictionaries for fast lookup

3. **Input Processing** (`check_input()` at line 504)
   - Validates input file format
   - Converts VCF to ANNOVAR format if needed

4. **ANNOVAR Annotation** (`check_annovar_result()` at line 535)
   - Calls ANNOVAR perl scripts to annotate variants
   - Generates `*.hg19_multianno.txt` or `*.hg38_multianno.txt` files
   - Can be skipped if annotation already exists

5. **Variant Interpretation** (`my_inter_var()` at line 1807)
   - Main interpretation loop for each variant
   - Evaluates each of the 28 ACMG criteria using individual check functions
   - Merges automated criteria with user-provided evidence

6. **Classification** (`classfy()` at line 733)
   - Combines all evidence codes following ACMG rules
   - Assigns final classification (pathogenic/likely pathogenic/VUS/likely benign/benign)
   - Outputs results to `*.intervar` file

### Evidence Checking Functions

Each ACMG criterion has a dedicated function (lines 858-1644):
- **Very Strong**: `check_PVS1()` - null variant in LOF gene
- **Strong**: `check_PS1()` through `check_PS4()` - same AA change, de novo, functional studies, prevalence
- **Moderate**: `check_PM1()` through `check_PM6()` - domain, frequency, trans, protein length, AA change, assumed de novo
- **Supporting**: `check_PP1()` through `check_PP5()` - cosegregation, missense, prediction, phenotype, pathogenic report
- **Stand-alone Benign**: `check_BA1()` - high frequency
- **Strong Benign**: `check_BS1()` through `check_BS4()` - frequency, healthy controls, splicing, non-segregation
- **Supporting Benign**: `check_BP1()` through `check_BP7()` - truncating, trans, in-frame, prediction, disorder, silent, synonymous

### Key Data Structures

Global dictionaries loaded from `intervardb/` (initialized lines 94-115):
- `lof_genes_dict` - Loss of function intolerant genes (PVS1)
- `aa_changes_dict` - Pathogenic amino acid changes (PS1)
- `domain_benign_dict` - Protein domains with benign variants (PM1)
- `mim2gene_dict` / `mim2gene_dict2` - OMIM gene mappings (PM2)
- `PP2_genes_dict` / `BP1_genes_dict` - Missense gene lists
- `PS4_snps_dict` - High odds-ratio variants
- `BS2_snps_dict` - Recessive/dominant healthy controls
- `knownGeneCanonical_dict` - Canonical gene transcripts

## Dependencies

### External Tools
- **Python** ≥ 2.6.6 (supports both Python 2 and 3)
- **ANNOVAR** ≥ 2016-02-01 (required for annotation step)
  - Scripts: `table_annovar.pl`, `convert2annovar.pl`, `annotate_variation.pl`
  - Database location configurable via `database_locat` parameter

### ANNOVAR Databases (configured in config.ini line 62)
Required databases: refGene, esp6500siv2_all, 1000g2015aug, avsnp147, dbnsfp42a, clinvar_20210501, gnomad_genome, dbscsnv11, rmsk, ensGene, knownGene

### InterVar Datasets (in intervardb/ directory)
Must download separately from OMIM (http://www.omim.org/downloads):
- `mim2gene.txt` - OMIM gene-phenotype relationships (REQUIRED)

Build-specific files (hg19/hg38 suffixes):
- `PVS1.LOF.genes.{buildver}` - Loss of function genes
- `PM1_domains_with_benigns.{buildver}` - Protein domains
- `PS1.AA.change.patho.{buildver}` - Pathogenic AA changes
- `PS4.variants.{buildver}` - High prevalence variants
- `BS2_hom_het.{buildver}` - Homozygous/heterozygous controls
- `PP2.genes.{buildver}` / `BP1.genes.{buildver}` - Gene lists
- `knownGeneCanonical.txt.{buildver}` - Canonical transcripts

Non-build-specific files:
- `mim_recessive.txt`, `mim_domin.txt`, `mim_adultonset.txt`, `mim_pheno.txt`, `mim_orpha.txt`
- `orpha.txt.utf8` - Orphanet disease database

## Configuration

Primary config file: `config.ini`

Key sections:
- `[InterVar]` - Main settings (buildver, input/output paths, database locations, cutoffs)
- `[InterVar_Bool]` - Boolean flags (onetranscript, otherinfo)
- `[Annovar]` - ANNOVAR paths and database settings
- `[Other]` - Version info

Important parameters:
- `buildver` - Genome build (hg19 or hg38)
- `disorder_cutoff` - Allele frequency cutoff for BS1 (default: 0.01)
- `database_intervar` - Path to InterVar datasets
- `database_locat` - Path to ANNOVAR humandb
- `evidence_file` - Optional user-provided evidence

## Input Formats

### ANNOVAR Input (AVinput)
Tab-delimited: Chr, Start, End, Ref, Alt, [optional_info]
```
1   948921   948921   T   C   comments: rs15842
```

### VCF Format
Standard VCF with single sample (VCF) or multiple samples (VCF_m)

### User Evidence File
Tab-delimited with custom evidence codes:
```
Chr   Pos   Ref_allele   Alt_allele   PM1=1;BS2=1;BP3=0;PS5=1;grade_PM1=1
```

Evidence codes: PS5, PM7, PP6, BS5, BP8 (additional criteria)
Grade adjustments: grade_XX=1 (Strong), =2 (Moderate), =3 (Supporting)

## Output Files

Main output: `{output_prefix}.{buildver}_multianno.txt.intervar`

Contains all input columns plus:
- Individual evidence codes (PVS1, PS1-4, PM1-6, PP1-5, BA1, BS1-4, BP1-7)
- Final classification
- InterVar reasoning

## Common Issues

1. **OMIM file missing**: Download `mim2gene.txt` from http://www.omim.org/downloads (line 214-216)
2. **Outdated OMIM files**: Use files generated >= 2016-09 (README line 17)
3. **Old ANNOVAR version**: Ensure ANNOVAR >= 2016-02-01 (line 52)
4. **Database files not found**: Check buildver suffix matches (hg19/hg38)
5. **Disorder cutoff warnings**: Default 0.005 for disease alleles, 0.01 configurable (line 42, 2084-2086)

## Code Style Notes

- Python 2/3 compatible using conditional imports (lines 60-63)
- Global state used extensively for loaded datasets
- Functions named after ACMG criteria (e.g., `check_PVS1`, `check_PS1`)
- Evidence results stored as integers: 0 (not met), 1 (met)
- Classification logic follows ACMG combinatorial rules in `classfy()` function
