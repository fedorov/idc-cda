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

<a name="collection-level-queries"></a>
### Collection-level queries: `project_short_name = <idc_collection_id>`

**You do not need to map collection IDs by hand.** CDA indexes the IDC `collection_id` itself as a
`project_short_name`, alongside the GDC project ID, the program name and the dbGaP accession. A
subject query with `ADD_COLUMNS` shows all of them at once:

```python
get_subject_data(match_all=['upstream_id = TCGA-DD-A4NO'],
                 add_columns=['project_short_name'])
# project_short_name -> ['phs000178', 'tcga', 'TCGA', 'tcga_lihc', 'TCGA-LIHC']
#                        dbGaP        program      IDC collection  GDC project
```

Every one of those is a valid filter value, so `project_short_name = tcga_lihc` selects exactly the
CDA subjects IDC holds imaging for in that collection.

This makes exhaustive cross-node analysis cheap: **one call per collection instead of one call per
patient**, with no sampling.

```python
summarize_subjects(match_all=['project_short_name = cptac_ccrcc'],
                   return_data_as='dict')
# {'number_of_matching_subjects': 233,
#  'number_of_files_related_to_matching_subjects': 35365,
#  'data_source': {'IDC only': 1, 'IDC and GDC': 1,
#                  'GDC and PDC and IDC': 121, 'GDC and PDC and IDC and GC': 110}}
```

**Verified 1:1.** Across all 46 TCGA and CPTAC collections, the CDA subject count returned by
`project_short_name = <collection_id>` equals the `idc-index` distinct-`PatientID` count for that
collection — 13,221 patients, zero mismatches. [Notebook 03](../notebooks/03_cross_node_analysis.ipynb)
checks this each run and prints any collection that disagrees.

`idc-index` 0.12.5 and CDA's June 2026 release both carry IDC v24, so this comparison is like-for-like.
That alignment is incidental — re-run the check rather than assuming it holds after a release on either
side.

**Two caveats:**

1. **Parse the `data_source` buckets by set membership, not by string.** Node order within a label is
   not stable between queries — both `'PDC and IDC and GC'` and `'IDC and GDC and PDC'` occur. Over
   REST the keys are `pdc_gc_idc_exclusive`-style instead; see
   [cda_overview.md](cda_overview.md#raw-response-shape).
2. **`/summary/file` scoped by `project_short_name` counts only that project's own files.** For
   `tcga_lihc` that is 3,457 DICOM files, all IDC — the genomic files for those same subjects belong
   to the GDC project. To reach files across nodes, go through `subject_id` (Step 4 below).

## Cross-Node Data Availability

### IDC Scale
- IDC v24: 176 collections, 85,362 patients, 1.03M series
- 32 TCGA collections (11,050 patients), 14 CPTAC collections (2,171 patients),
  5 HTAN collections (69 patients)
- CDA's June 2026 release indexes IDC **v24**, matching `idc-index` 0.12.5; check `/release_metadata/`
  for CDA's current snapshot, which lags the live IDC release

### Exact cross-node coverage (IDC v24, CDA June 2026 release)

Complete counts over all 13,221 TCGA + CPTAC patients — not sampled. Produced by
[notebook 03](../notebooks/03_cross_node_analysis.ipynb).

| Program | Collections | IDC patients | Also in GDC | Also in PDC | Also in GC |
|---------|------------:|-------------:|------------:|------------:|-----------:|
| TCGA    | 32 | 11,050 | 11,045 (100%) | 369 (3%) | 1,936 (18%) |
| CPTAC   | 14 | 2,171 | 1,623 (75%) | 1,636 (75%) | 1,093 (50%) |
| **Total** | **46** | **13,221** | **12,668 (96%)** | **2,005 (15%)** | **3,029 (23%)** |

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

### Step 2: Query CDA

```python
from cdapython import get_subject_data, summarize_subjects

# Single patient
subject = get_subject_data(match_all=['upstream_id = TCGA-DD-A4NO'])
print(subject['data_source'].iloc[0])  # ['GDC', 'IDC']

# A whole IDC collection, exactly — preferred over batching patient IDs
summary = summarize_subjects(match_all=['project_short_name = tcga_lihc'],
                             return_data_as='dict')

# An arbitrary ID list — chunk at ~200 filters (see cda_overview.md#known-issues)
subjects = get_subject_data(match_any=[f'upstream_id = {p}' for p in patient_ids[:200]])
```

### Step 3: Analyze Data Source Distribution

```python
# Per-subject, from get_subject_data
sources = subjects['data_source'].apply(lambda x: tuple(sorted(x)))
print(sources.value_counts())

# Or aggregate, from summarize_subjects — parse by set membership, not string
buckets = {frozenset(k.replace(' only', '').split(' and ')): v
           for k, v in summary['data_source'].items() if v}
# {frozenset({'IDC', 'GDC'}): 377}  <- all 377 tcga_lihc subjects have both
```

### Step 4: Get Detailed File Information

```python
from cdapython import get_file_data

# Use subject_id (not upstream_id) for file queries
# exclude_columns=['anatomic_site'] works around a cdapython 2.0.14 KeyError
# — see cda_overview.md#known-issues
files = get_file_data(match_all=['subject_id = TCGA.TCGA-DD-A4NO'],
                      exclude_columns=['anatomic_site'])
exploded = files.explode('data_source')
breakdown = exploded.groupby(['data_source', 'category']).agg(
    files=('file_id', 'count'),
    size_MB=('size', lambda x: round(x.sum() / 1e6, 1))
)
```

## Key Observations

1. **TCGA has near-total genomic coverage**: 11,045 of 11,050 TCGA patients in IDC also have GDC data
2. **CPTAC adds proteomics**: 75% of CPTAC patients in IDC have both GDC and PDC data. TCGA
   proteomics is rarer but not negligible — 369 patients (3%), which patient sampling misses entirely
3. **Not all CPTAC IDC collections have GDC/PDC data**: Some newer CPTAC collections (e.g., `cptac_sar`, `cptac_hnscc`) appear in IDC but may not yet be in GDC/PDC
4. **HTAN is emerging**: Only 68 patients in IDC, cross-node links via CDA are still developing
5. **Other IDC collections**: Non-TCGA/CPTAC collections (NLST, CBIS-DDSM, etc.) generally have only IDC data in CRDC; their data may exist in other repositories outside CRDC

## CDA Query Performance Notes

- Single subject lookups: ~1-2 seconds
- Collection summary (`project_short_name = ...`): ~1-3 seconds for any collection size — all 46
  TCGA + CPTAC collections sweep in about two minutes
- Batch `match_any` scales with filter count: ~3 s at 50 filters, ~12 s at 200; **200 is near the
  request-size ceiling** (see [cda_overview.md](cda_overview.md#known-issues))
- The service occasionally stalls for minutes under load and then recovers. Wrap sweeps in a retry
  rather than assuming a slow response means a large query — the same call that took 3,000 s during
  one sweep returned in 0.8 s on retry
- File queries can be large (100+ files per subject for TCGA)
