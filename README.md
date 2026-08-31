# IDC-CDA: Cross-Node Data Discovery for Cancer Imaging

Exploring how to use the [CRDC Cancer Data Aggregator (CDA)](https://datacommons.cancer.gov/cancer-data-aggregator) with [NCI Imaging Data Commons (IDC)](https://portal.imaging.datacommons.cancer.gov/) to discover what non-imaging data (genomic, proteomic, clinical) is available from other CRDC nodes for cases that have imaging data in IDC.

## Key Findings

**Of the 13,221 patients in TCGA and CPTAC collections in IDC v24** — exact counts over every patient, not sampled:

| Program | Collections | IDC Patients | With GDC (Genomic) Data | With PDC (Proteomic) Data | With GC (Other CRDC) Data |
|---------|------------:|-------------|------------------------|--------------------------|--------------------------|
| TCGA | 32 | 11,050 | 11,045 (100%) | 369 (3%) | 1,936 (18%) |
| CPTAC | 14 | 2,171 | 1,623 (75%) | 1,636 (75%) | 1,093 (50%) |
| **Total** | **46** | **13,221** | **12,668 (96%)** | **2,005 (15%)** | **3,029 (23%)** |

The link that makes this cheap: **CDA indexes each IDC `collection_id` as a `project_short_name`**, so one
`summarize_subjects(match_all=['project_short_name = tcga_lihc'])` call returns complete cross-node counts
for a whole collection. Across all 46 collections the CDA subject count matches the `idc-index` patient
count exactly, with zero mismatches.

**Data available from other CRDC nodes for IDC imaging cases:**
- **GDC (Genomic Data Commons):** Sequencing reads (WGS, WXS, RNA-seq), copy number variation, DNA methylation, transcriptome profiling, somatic mutations, clinical supplements
- **PDC (Proteomic Data Commons):** Mass spectrometry proteomics, protein abundance quantification
- **CDA enrichment tables:** Diagnosis/staging (observation), treatments, mutations, cross-node identifiers

## Repository Contents

```
.
├── README.md
├── LICENSE                          # Apache-2.0
├── pyproject.toml                   # Dependencies: idc-index, cdapython, pandas, jupyter, etc.
├── .python-version                  # Python 3.11
├── uv.lock                         # Reproducible dependency lock
├── notebooks/
│   ├── 01_idc_overview.ipynb        # IDC data landscape: 176 collections, modalities, cancer types
│   ├── 02_cda_exploration.ipynb     # CDA API discovery: tables, columns, query patterns
│   ├── 03_cross_node_analysis.ipynb # Cross-referencing IDC cases in CDA with visualizations
│   └── 04_cda_rest_api.ipynb        # Same queries over plain HTTP, without cdapython
├── developer/
│   ├── cda_overview.md              # CDA data model, Python API reference, query examples
│   └── idc_cda_integration.md       # Identifier mapping, cross-node workflows, observations
└── data/                            # Generated intermediate results (gitignored)
```

### Notebooks

**[01_idc_overview.ipynb](notebooks/01_idc_overview.ipynb)** — IDC data landscape overview. Queries IDC (v24, per `idc-index` 0.12.5) for collection-level statistics: patient counts, modalities, sizes, cancer types. Identifies TCGA (32 collections, 11,050 patients), CPTAC (14 collections, 2,171 patients), and HTAN (5 collections, 69 patients) as cross-node candidates. Extracts PatientIDs for downstream analysis.

**[02_cda_exploration.ipynb](notebooks/02_cda_exploration.ipynb)** — CDA API exploration. Discovers CDA's data model: 7 tables (subject, observation, file, treatment, mutation, project, upstream_identifiers). Tests query patterns, identifies `upstream_id` as the linking key between IDC PatientIDs and CDA subjects. Demonstrates that `data_source` column reveals which CRDC nodes have data per subject.

**[03_cross_node_analysis.ipynb](notebooks/03_cross_node_analysis.ipynb)** — Core cross-node analysis. For each of the 46 TCGA and CPTAC IDC collections, queries CDA with `project_short_name = <collection_id>` to get exact GDC/PDC/GC data availability for every patient — one call per collection, about two minutes for the full sweep. Checks CDA subject counts against `idc-index` patient counts and reports any collection that disagrees. Produces a heatmap and bar chart of cross-node coverage, plus deep-dive file breakdowns per node for example subjects.

**[04_cda_rest_api.ipynb](notebooks/04_cda_rest_api.ipynb)** — The same CDA queries over plain HTTP with only `requests` — no `cdapython`, no `cda-client`, no pinned pandas. Covers the seven REST endpoints, the raw response shape (per-node booleans rather than `cdapython`'s `data_source` list), pagination, the `query_sql` field every response carries, `ADD_COLUMNS` and `EXTERNAL_REFERENCE`, `SEARCH_LIST` ontology keyword search (available over REST but absent from `cda-client` 2.0.1), the undocumented ~8 KB request-size limit, and equivalent `curl` and R snippets. Useful for non-Python workflows, dependency-free environments, and for sidestepping the `cdapython` bugs noted below.

### Developer Documentation

**[developer/cda_overview.md](developer/cda_overview.md)** — Comprehensive CDA reference: data model, table schemas, Python API (`cdapython`) function signatures, query parameters, filter syntax, and code examples.

**[developer/idc_cda_integration.md](developer/idc_cda_integration.md)** — How to link IDC and CDA: identifier mapping (PatientID to upstream_id to subject_id), step-by-step cross-node workflow, collection ID mapping, performance notes.

## Setup

Requires [uv](https://docs.astral.sh/uv/) (system-installed).

```bash
cd idc-cda
uv venv
uv sync
source .venv/bin/activate
jupyter notebook
```

## How It Works

1. **IDC** (`idc-index` package) provides a local SQL-queryable index of all imaging data — no authentication needed
2. **CDA** (`cdapython` package) queries a unified metadata store across CRDC nodes — no authentication needed
3. **Linking:** IDC's `PatientID` matches CDA's `upstream_id` for TCGA (barcode like `TCGA-DD-A4NO`) and CPTAC (ID like `C3N-02973`) collections
4. **CDA's `data_source` column** reveals which CRDC nodes have data for each subject (e.g., `['GDC', 'IDC', 'PDC']`). Note this is a `cdapython` client-side reshape — the API itself returns one boolean per node (`subject_data_at_gdc`, `subject_data_at_idc`, …)
5. **Collection-level shortcut:** `project_short_name = <idc_collection_id>` selects exactly the CDA subjects IDC has imaging for in that collection, making exhaustive analysis a single call per collection
6. **No client library required.** CDA is a public FastAPI service at `https://cda.datacommons.cancer.gov`; [notebook 04](notebooks/04_cda_rest_api.ipynb) does everything above with `requests` alone

## Limitations

- **Undocumented ~8 KB request-size limit.** A gateway in front of the CDA API rejects request bodies over roughly 8 KB with a bare HTTP 403 and an HTML page — not a JSON error, and absent from the OpenAPI spec. Measured: 260 `MATCH_SOME` filters succeed, 300 fail. This affects `cdapython` identically. Batched ID lists must be chunked at ~200 filters; collection-level queries avoid the problem entirely.
- **Variable API latency.** Collection summaries normally return in 1-3 seconds, but the service occasionally stalls for minutes under load and then recovers — during one sweep a call took 3,000 s and the same call returned in 0.8 s on retry. Notebook 03 wraps its sweep in a retry.
- **Not all IDC collections have cross-node data.** Only TCGA, CPTAC, and HTAN collections have expected counterparts in GDC/PDC. The remaining 125 IDC collections (~72K patients) are from independent imaging studies with no corresponding data in other CRDC nodes.
- **CDA data currency.** CDA aggregates from periodic snapshots of each CRDC node. There may be a lag between when data appears in GDC/PDC/IDC and when CDA indexes it.
- **Identifier matching is program-specific.** The `upstream_id = <PatientID>` and `project_short_name = <collection_id>` approaches are verified for the 46 TCGA and CPTAC collections. For other programs (HTAN, NLST, etc.), identifier formats differ and may not match directly.
- **No file download through CDA.** CDA provides metadata only. Actual data download requires using each node's tools: `idc-index` for imaging, GDC Data Transfer Tool for genomics, PDC tools for proteomics.
- **`cdapython` API stability.** The `cdapython` package is under active development, and two bugs in v2.0.14 currently affect these notebooks:
  - `column_values()` intermittently crashes with `"Unexpected data type 'str' received"` due to a pandas `StringDtype` incompatibility. Workaround: `pd.set_option('future.infer_string', False)`.
  - `get_file_data()` fails outright with `KeyError: 'anatomic_site_containing_terms'` against the CDA June 2026 release, because the client cannot type the ontology-expansion columns the API returns. Workaround: `exclude_columns=['anatomic_site']`.

  Both are client-side only — the identical queries succeed over the REST API ([notebook 04](notebooks/04_cda_rest_api.ipynb)). See [developer/cda_overview.md](developer/cda_overview.md#known-issues) for details.
- **Data version alignment.** `idc-index` 0.12.5 and CDA's June 2026 release both carry IDC **v24**, so notebook 03's collection-level comparison is like-for-like. That alignment is incidental, not guaranteed: CDA also indexes GDC Data Release 45.0 and PDC Data Release 6.1, and lags each node's live portal by varying amounts. Notebook 03 re-checks the IDC/CDA agreement on every run rather than assuming it; check `release_metadata()` (or `GET /release_metadata/`) for CDA's current snapshot.
- **License awareness.** While the code in this repository is Apache-2.0, the underlying cancer data has its own licenses (mostly CC BY 4.0, some CC BY-NC). Always check `license_short_name` in IDC before using data.

## How This Repository Was Created

This repository was generated on **February 16, 2026** using [Claude Code](https://claude.ai/claude-code) (Claude Opus 4.6) with the [IDC Claude skill](https://github.com/ImagingDataCommons/idc-claude-skill) for Imaging Data Commons expertise.

**Tools and versions used:**
- **Claude Code** — AI-assisted code generation, CDA API exploration, and documentation
- **idc-index 0.11.9** (IDC data version v23) — local IDC metadata index and download client (upgraded to 0.12.5 / IDC v24 in the follow-up session below)
- **cdapython 2.0.14** (cda-client 2.0.1) — Python client for the CDA REST API
- **uv** — Python package and environment manager
- **Python 3.11**, pandas, matplotlib, seaborn, Jupyter

**Follow-up session (August 31, 2026)** — upgraded `idc-index` 0.11.9 → **0.12.5** (IDC v23 → **v24**), which brought the local index in line with CDA's own IDC snapshot and required adapting notebook 01 to `collections_index`'s renamed snake_case columns. Added [notebook 04](notebooks/04_cda_rest_api.ipynb) covering the CDA REST API directly, and replaced notebook 03's patient sampling with exact per-collection queries after discovering that CDA indexes IDC `collection_id` values as `project_short_name`. That session also surfaced the `get_file_data()` bug documented above, corrected the CDA base URL in the developer docs (`/api/` 404s), and documented the ~8 KB request-size limit.

All figures in this README were regenerated on IDC v24 with the CDA June 2026 release.

**Prompting process:** The initial repository was created in a single Claude Code session guided by [Andrey Fedorov](https://github.com/fedorov). The process involved the following prompts:

1. **Initial prompt** defined the goal: explore how to use CDA with IDC, analyze what is available from CDA, and prepare an initial summary of what is possible in terms of finding data from other CRDC components for IDC cases. The `/imaging-data-commons` skill was invoked for IDC expertise.
2. **Environment setup** — instructed to use system `uv` for the Python virtual environment.
3. **Documentation and packaging** — requested a `developer/` folder with analysis documentation, a top-level README with limitations, and an Apache-2.0 license.
4. **Refinements** — `.gitignore` additions (`.claude/`, keeping `uv.lock`), this provenance section, and attribution.

Claude Code autonomously handled: researching the CDA API (discovering tables, column schemas, query patterns), empirically testing cross-node identifier linking (verifying that IDC `PatientID` maps to CDA `upstream_id`), iteratively building and executing notebooks (including fixing a timeout by reducing sample sizes), and writing developer documentation. All notebook outputs reflect live queries against the IDC index and CDA API as of the date above.

## License

This repository is licensed under the [Apache License 2.0](LICENSE).

Note: The cancer imaging and genomic data accessed through IDC and CDA has its own licensing terms. Most IDC data is CC BY 4.0 (allows commercial use with attribution); approximately 3% is CC BY-NC (non-commercial only). Always verify data licenses before use.
