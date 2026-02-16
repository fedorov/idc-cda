# Cancer Data Aggregator (CDA) — Developer Reference

## What is CDA?

The Cancer Data Aggregator (CDA) is a service of the NCI Cancer Research Data Commons (CRDC) that provides a unified search engine to query and aggregate metadata across multiple cancer research repositories. It combines standardized, harmonized, and cross-referenced descriptive terms from multiple data sources into unified records.

**Key facts:**
- Indexes 42M+ files, 139K+ subjects, 821K+ specimens
- No authentication required for queries
- REST API: `https://cda.datacommons.cancer.gov/api/`
- Python client: `cdapython` (PyPI: `cda-client`)

## Data Sources Aggregated

| Source | Full Name | Data Types | Scale (upstream IDs) |
|--------|-----------|-----------|---------------------|
| **GDC** | Genomic Data Commons | Sequencing, methylation, copy number, mutations, clinical | ~100K IDs |
| **IDC** | Imaging Data Commons | CT, MR, PET, pathology images | ~240K IDs |
| **GC** | Genomic Commons / Cancer Data Services | Additional datasets | ~145K IDs |
| **PDC** | Proteomic Data Commons | Mass spectrometry proteomics | ~11K IDs |
| **ICDC** | Integrated Canine Data Commons | Canine cancer research | ~3K IDs |
| **dbGaP** | Database of Genotypes and Phenotypes | Genotype-phenotype studies | ~200 IDs |

## CDA Data Model

### Tables

| Table | Description | Key Columns |
|-------|-------------|-------------|
| `subject` | Core demographics per harmonized subject | `subject_id`, `cause_of_death`, `ethnicity`, `race`, `species`, `year_of_birth`, `year_of_death` |
| `observation` | Diagnoses, staging, morphology | `age_at_observation`, `diagnosis`, `grade`, `morphology`, `observed_anatomic_site`, `sex`, `stage`, `vital_status` |
| `file` | Files across all CRDC nodes | `file_id`, `access`, `anatomic_site`, `category`, `drs_uri`, `file_type`, `format`, `size`, `tumor_vs_normal` |
| `treatment` | Therapeutic interventions | `therapeutic_agent`, `treatment_anatomic_site`, `treatment_type` |
| `mutation` | Somatic mutations (GDC) | `hugo_symbol`, `chromosome`, `variant_type`, `mutation_status`, `one_consequence`, `case_barcode` |
| `project` | Project/collection groupings | `project_id`, `project_name`, `project_short_name`, `project_type` |
| `upstream_identifiers` | Cross-reference to source IDs | `upstream_id`, `upstream_field`, `upstream_source` |

### Key Identifier Fields in `upstream_identifiers`

These are the `upstream_field` values and what they map:

| upstream_field | upstream_source | Description | Count |
|---------------|-----------------|-------------|-------|
| `dicom_all.PatientID` | IDC | DICOM Patient ID (= IDC PatientID) | ~79,650 |
| `dicom_all.idc_case_id` | IDC | IDC internal case ID | ~79,649 |
| `case.submitter_id` | GDC | GDC submitter-assigned case ID (= TCGA barcode for TCGA) | ~50,213 |
| `case.case_id` | GDC | GDC UUID | ~51,299 |
| `Case.case_submitter_id` | PDC | PDC submitter-assigned case ID | ~4,982 |
| `Case.case_id` | PDC | PDC UUID | ~5,047 |
| `participant.participant_id` | GC | CDS participant ID | ~60,776 |

## Python API (`cdapython`)

### Installation

```bash
pip install cdapython
```

### Core Functions

```python
from cdapython import (
    tables,              # List available tables
    columns,             # List columns (optionally filter by table/column)
    column_values,       # Get distinct values for a column
    release_metadata,    # CDA release statistics
    summarize_subjects,  # Aggregate subject counts
    summarize_files,     # Aggregate file counts
    get_subject_data,    # Retrieve subject records
    get_file_data,       # Retrieve file records
)
```

### Query Parameters

All query functions (`get_subject_data`, `get_file_data`, `summarize_*`) accept:

| Parameter | Type | Description |
|-----------|------|-------------|
| `match_all` | `list[str]` | All filters must match (AND). Format: `"COLUMN OP VALUE"` |
| `match_any` | `list[str]` | Any filter can match (OR). Format: `"COLUMN OP VALUE"` |
| `match_from_file` | `dict` | Batch matching from a CSV file |
| `data_source` | `str` | Filter by CRDC node: `'GDC'`, `'PDC'`, `'IDC'`, `'GC'` |
| `add_columns` | `list[str]` | Include additional columns in results |
| `exclude_columns` | `list[str]` | Exclude columns from results |

**Filter operators:** `=`, `!=`, `<`, `>`, `<=`, `>=`
**Wildcards:** `*` for text matching (e.g., `upstream_id = TCGA-DD-*`)
**NULL:** Use `NULL` keyword for missing values

### Key Behaviors

1. **`get_subject_data` returns:** DataFrame with `subject_id`, demographics, `data_source` (list of nodes), `upstream_id` (list of all IDs)
2. **`get_file_data` returns:** DataFrame with file metadata. The `data_source` and `subject_id` columns are lists (a file may relate to multiple subjects/sources)
3. **`upstream_id` filter:** Works in `get_subject_data` but **NOT** in `get_file_data`. For files, use `subject_id` from the subject query
4. **Batch queries:** Use `match_any` with multiple filters for batch lookups. CDA handles the OR logic server-side

### Example: Cross-Node Lookup

```python
from cdapython import get_subject_data, get_file_data

# 1. Find subject by IDC PatientID (= TCGA barcode for TCGA data)
subject = get_subject_data(match_all=['upstream_id = TCGA-DD-A4NO'])
# subject['data_source'] → ['GDC', 'IDC'] — tells you which nodes have data

# 2. Get all files across nodes using subject_id
sid = subject['subject_id'].iloc[0]  # e.g., 'TCGA.TCGA-DD-A4NO'
files = get_file_data(match_all=[f'subject_id = {sid}'])

# 3. Breakdown by node and category
exploded = files.explode('data_source')
breakdown = exploded.groupby(['data_source', 'category']).size()
```

### Example: Batch Query

```python
# Query multiple patients at once using match_any
result = get_subject_data(match_any=[
    'upstream_id = TCGA-DD-A4NO',
    'upstream_id = TCGA-DD-A4NQ',
    'upstream_id = TCGA-05-4244'
])

# Or use wildcards
result = get_subject_data(match_all=['upstream_id = TCGA-DD-*'])
```

## Known Issues

### `column_values()` crash with pandas StringDtype (cdapython v2.0.14)

**Symptom:** `column_values()` fails with:
```
CRITICAL column_values(): ERROR: Unexpected data type `str` received; aborting.
Please report this event to the CDA development team.
```

**Root cause:** In `cdapython/discover.py` lines 1094-1122, the dtype dispatch only handles `float64`, `object`, and `bool`. When pandas infers string columns as `StringDtype` (`'string'`) instead of `object` — which can happen intermittently depending on the CDA API response serialization — the code falls through to the error branch.

The bug is in this logic:
```python
if result_dataframe[column].dtype == 'float64':
    ...
elif result_dataframe[column].dtype == 'object':  # ← misses 'string' dtype
    ...
elif result_dataframe[column].dtype == 'bool':
    ...
else:
    log.critical(...)  # ← crashes here when dtype is 'string'
```

**Workaround:** Set this option before importing or using `cdapython`:
```python
import pandas as pd
pd.set_option('future.infer_string', False)
```

This forces pandas to use `object` dtype for all string columns, avoiding the unhandled `StringDtype` path.

**Affected versions:** `cdapython` 2.0.14 with `pandas` >= 2.1 (which introduced `StringDtype` inference). The issue is intermittent — it depends on how the CDA API serializes its response for a given query.

**Upstream fix:** The `elif` at line 1103 should be:
```python
elif result_dataframe[column].dtype == 'object' or str(result_dataframe[column].dtype) == 'string':
```
