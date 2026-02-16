# IDC + CDA Integration — Developer Reference

## Overview

This document describes how to link Imaging Data Commons (IDC) imaging data with other CRDC nodes (GDC genomics, PDC proteomics) via the Cancer Data Aggregator (CDA).

## Identifier Mapping

### How Identifiers Connect Across CRDC Nodes

```
IDC PatientID ──────┐
                     ├──→ CDA upstream_id ──→ CDA subject_id ──→ All CRDC data
GDC submitter_id ────┘
```

For **TCGA** collections:
- IDC `PatientID` = TCGA case barcode (e.g., `TCGA-DD-A4NO`)
- GDC `case.submitter_id` = same TCGA case barcode
- CDA merges these into a single `subject_id` (e.g., `TCGA.TCGA-DD-A4NO`)

For **CPTAC** collections:
- IDC `PatientID` = CPTAC participant ID (e.g., `C3N-02973`)
- GDC `case.submitter_id` = same participant ID
- PDC `Case.case_submitter_id` = same participant ID
- CDA merges into `subject_id` (e.g., `CPTAC.C3N-02973`)

### Collection ID Mapping

IDC uses lowercase with underscores; GDC uses uppercase with hyphens:

| IDC `collection_id` | GDC `project.project_id` | Cancer Type |
|---------------------|-------------------------|-------------|
| `tcga_luad` | `TCGA-LUAD` | Lung Adenocarcinoma |
| `tcga_brca` | `TCGA-BRCA` | Breast Cancer |
| `cptac_ucec` | `CPTAC-3` | Uterine/Endometrial |

## Cross-Node Data Availability

### IDC Scale (v23)
- 161 collections, ~79,500 patients, ~994K series, ~95 TB
- 32 TCGA collections (~11,050 patients)
- 13 CPTAC collections (~2,150 patients)
- 4 HTAN collections (~68 patients)

### What Other Nodes Have

**GDC (Genomic Data Commons)** — available for nearly all TCGA and most CPTAC cases:
- Sequencing Reads (BAM): WGS, WXS, RNA-seq
- Copy Number Variation
- DNA Methylation
- Simple Nucleotide Variation (somatic mutations)
- Transcriptome Profiling (gene expression)
- Clinical Supplements (BCR biotab)
- Biospecimen data

**PDC (Proteomic Data Commons)** — available for many CPTAC cases:
- Mass spectrometry proteomics
- Protein abundance quantification
- Phosphoproteomics
- Glycoproteomics

### Example File Breakdown for a TCGA-LIHC Subject

```
GDC:
  Sequencing Reads         17 files, 560 GB
  Biospecimen              15 files,   2 GB
  Copy Number Variation    16 files, 500 MB
  Simple Nucleotide Var.   31 files,  71 MB
  DNA Methylation           3 files,  29 MB
  Transcriptome Profiling   4 files,   7 MB

IDC:
  CT images                18 series, 938 MB
  MR images                50 series,   2 GB
  Segmentations             6 series,   9 MB
  Slide Microscopy           3 series,   2 GB
```

## Workflow: Finding Cross-Node Data for IDC Cases

### Step 1: Get Patient IDs from IDC

```python
from idc_index import IDCClient

idc = IDCClient()
patients = idc.sql_query("""
    SELECT DISTINCT PatientID, collection_id
    FROM index
    WHERE collection_id LIKE 'tcga_%'
""")
```

### Step 2: Query CDA for Each Patient

```python
from cdapython import get_subject_data

# Single patient
subject = get_subject_data(match_all=['upstream_id = TCGA-DD-A4NO'])
print(subject['data_source'].iloc[0])  # ['GDC', 'IDC']

# Batch: all patients from a collection prefix
subjects = get_subject_data(match_all=['upstream_id = TCGA-DD-*'])
```

### Step 3: Analyze Data Source Distribution

```python
# For each subject, which CRDC nodes have data?
sources = subjects['data_source'].apply(lambda x: tuple(sorted(x)))
print(sources.value_counts())
# ('GDC', 'IDC')    151  ← all TCGA-LIHC subjects have both GDC + IDC
```

### Step 4: Get Detailed File Information

```python
from cdapython import get_file_data

# Use subject_id (not upstream_id) for file queries
files = get_file_data(match_all=['subject_id = TCGA.TCGA-DD-A4NO'])
exploded = files.explode('data_source')
breakdown = exploded.groupby(['data_source', 'category']).agg(
    files=('file_id', 'count'),
    size_MB=('size', lambda x: round(x.sum() / 1e6, 1))
)
```

## Key Observations

1. **TCGA has the richest cross-node coverage**: Nearly all TCGA patients in IDC also have GDC genomic data
2. **CPTAC adds proteomics**: CPTAC subjects may have data in all three: IDC (imaging) + GDC (genomics) + PDC (proteomics)
3. **Not all CPTAC IDC collections have GDC/PDC data**: Some newer CPTAC collections (e.g., `cptac_sar`, `cptac_hnscc`) appear in IDC but may not yet be in GDC/PDC
4. **HTAN is emerging**: Only 68 patients in IDC, cross-node links via CDA are still developing
5. **Other IDC collections**: Non-TCGA/CPTAC collections (NLST, CBIS-DDSM, etc.) generally have only IDC data in CRDC; their data may exist in other repositories outside CRDC

## CDA Query Performance Notes

- Single subject lookups: ~1-2 seconds
- Batch queries with `match_any` (20 filters): ~3-5 seconds
- Wildcard queries (`TCGA-DD-*`): ~2-4 seconds
- Rate limiting: Be respectful with large batch queries; add `time.sleep(0.5)` between batches
- File queries can be large (100+ files per subject for TCGA)
