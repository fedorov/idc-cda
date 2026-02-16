# IDC-CDA: Cross-Node Data Discovery for Cancer Imaging

Exploring how to use the [CRDC Cancer Data Aggregator (CDA)](https://datacommons.cancer.gov/cancer-data-aggregator) with [NCI Imaging Data Commons (IDC)](https://portal.imaging.datacommons.cancer.gov/) to discover what non-imaging data (genomic, proteomic, clinical) is available from other CRDC nodes for cases that have imaging data in IDC.

## Key Findings

**Of the ~13,200 patients in TCGA and CPTAC collections in IDC:**

| Program | IDC Patients | With GDC (Genomic) Data | With PDC (Proteomic) Data |
|---------|-------------|------------------------|--------------------------|
| TCGA | 11,050 | 11,050 (100%) | 68 (~1%) |
| CPTAC | 2,151 | 1,596 (74%) | 1,535 (71%) |
| **Total** | **13,201** | **12,646 (96%)** | **1,603 (12%)** |

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
│   ├── 01_idc_overview.ipynb        # IDC data landscape: 161 collections, modalities, cancer types
│   ├── 02_cda_exploration.ipynb     # CDA API discovery: tables, columns, query patterns
│   └── 03_cross_node_analysis.ipynb # Cross-referencing IDC cases in CDA with visualizations
├── developer/
│   ├── cda_overview.md              # CDA data model, Python API reference, query examples
│   └── idc_cda_integration.md       # Identifier mapping, cross-node workflows, observations
└── data/                            # Generated intermediate results (gitignored)
```

### Notebooks

**[01_idc_overview.ipynb](notebooks/01_idc_overview.ipynb)** — IDC data landscape overview. Queries IDC (v23) for collection-level statistics: patient counts, modalities, sizes, cancer types. Identifies TCGA (32 collections, 11K patients), CPTAC (13 collections, 2.2K patients), and HTAN (4 collections, 68 patients) as cross-node candidates. Extracts PatientIDs for downstream analysis.

**[02_cda_exploration.ipynb](notebooks/02_cda_exploration.ipynb)** — CDA API exploration. Discovers CDA's data model: 7 tables (subject, observation, file, treatment, mutation, project, upstream_identifiers). Tests query patterns, identifies `upstream_id` as the linking key between IDC PatientIDs and CDA subjects. Demonstrates that `data_source` column reveals which CRDC nodes have data per subject.

**[03_cross_node_analysis.ipynb](notebooks/03_cross_node_analysis.ipynb)** — Core cross-node analysis. For each TCGA and CPTAC IDC collection, samples patients and queries CDA to determine GDC/PDC data availability. Produces a heatmap and bar chart of cross-node coverage. Includes deep-dive file breakdowns showing specific data categories available per node for example subjects.

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
4. **CDA's `data_source` column** reveals which CRDC nodes have data for each subject (e.g., `['GDC', 'IDC', 'PDC']`)

## Limitations

- **Sampling-based estimates.** Cross-node availability percentages in Notebook 03 are based on 10 sampled patients per collection. Actual coverage may differ, though TCGA showed 100% GDC coverage across all 32 collections.
- **CDA API rate limits.** Large-scale queries (thousands of patients) require batching and may be slow (~1-3 seconds per API call). The notebooks sample rather than exhaustively query.
- **Not all IDC collections have cross-node data.** Only TCGA, CPTAC, and HTAN collections have expected counterparts in GDC/PDC. The remaining ~112 IDC collections (~66K patients) are from independent imaging studies with no corresponding data in other CRDC nodes.
- **CDA data currency.** CDA aggregates from periodic snapshots of each CRDC node. There may be a lag between when data appears in GDC/PDC/IDC and when CDA indexes it.
- **Identifier matching is program-specific.** The `upstream_id = <PatientID>` approach works reliably for TCGA and CPTAC. For other programs (HTAN, NLST, etc.), identifier formats differ and may not match directly.
- **No file download through CDA.** CDA provides metadata only. Actual data download requires using each node's tools: `idc-index` for imaging, GDC Data Transfer Tool for genomics, PDC tools for proteomics.
- **`cdapython` API stability.** The `cdapython` package is under active development. Function signatures and behaviors may change between releases. There is a known bug in v2.0.14 where `column_values()` intermittently crashes with `"Unexpected data type 'str' received"` due to a pandas `StringDtype` incompatibility. The notebooks include a workaround (`pd.set_option('future.infer_string', False)`). See [developer/cda_overview.md](developer/cda_overview.md#known-issues) for details.
- **IDC data version dependency.** Results are based on IDC v23. Future IDC releases may add collections or change PatientIDs.
- **License awareness.** While the code in this repository is Apache-2.0, the underlying cancer data has its own licenses (mostly CC BY 4.0, some CC BY-NC). Always check `license_short_name` in IDC before using data.

## How This Repository Was Created

This repository was generated on **February 16, 2026** using [Claude Code](https://claude.ai/claude-code) (Claude Opus 4.6) with the [IDC Claude skill](https://github.com/ImagingDataCommons/idc-claude-skill) for Imaging Data Commons expertise.

**Tools and versions used:**
- **Claude Code** — AI-assisted code generation, CDA API exploration, and documentation
- **idc-index 0.11.9** (IDC data version v23) — local IDC metadata index and download client
- **cdapython 2.0.14** (cda-client 2.0.1) — Python client for the CDA REST API
- **uv** — Python package and environment manager
- **Python 3.11**, pandas, matplotlib, seaborn, Jupyter

**Prompting process:** The entire repository was created in a single Claude Code session guided by [Andrey Fedorov](https://github.com/fedorov). The process involved the following prompts:

1. **Initial prompt** defined the goal: explore how to use CDA with IDC, analyze what is available from CDA, and prepare an initial summary of what is possible in terms of finding data from other CRDC components for IDC cases. The `/imaging-data-commons` skill was invoked for IDC expertise.
2. **Environment setup** — instructed to use system `uv` for the Python virtual environment.
3. **Documentation and packaging** — requested a `developer/` folder with analysis documentation, a top-level README with limitations, and an Apache-2.0 license.
4. **Refinements** — `.gitignore` additions (`.claude/`, keeping `uv.lock`), this provenance section, and attribution.

Claude Code autonomously handled: researching the CDA API (discovering tables, column schemas, query patterns), empirically testing cross-node identifier linking (verifying that IDC `PatientID` maps to CDA `upstream_id`), iteratively building and executing notebooks (including fixing a timeout by reducing sample sizes), and writing developer documentation. All notebook outputs reflect live queries against the IDC index and CDA API as of the date above.

## License

This repository is licensed under the [Apache License 2.0](LICENSE).

Note: The cancer imaging and genomic data accessed through IDC and CDA has its own licensing terms. Most IDC data is CC BY 4.0 (allows commercial use with attribution); approximately 3% is CC BY-NC (non-commercial only). Always verify data licenses before use.
