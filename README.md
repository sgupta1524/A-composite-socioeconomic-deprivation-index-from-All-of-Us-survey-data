# aou-sed-gwas

Individual-level Social and Economic Determinants (SED) index construction
from the **All of Us** Controlled Tier Dataset v7, for use as covariates in
downstream GWAS and PRS analyses.

## Overview

Constructs a polychoric-PCA-based individual deprivation index from five
AoU survey domains. All variables are encoded ordinally; **higher value = more deprived**.
PMI: Skip and PMI: Prefer Not To Answer are treated as `NA` throughout.

---

### Income — concept 1585375 — 9 levels

| Code | AoU answer string                    |
|-----:|--------------------------------------|
| 1    | Annual Income: more 200k             |
| 2    | Annual Income: 150k 200k             |
| 3    | Annual Income: 100k 150k             |
| 4    | Annual Income: 75k 100k              |
| 5    | Annual Income: 50k 75k               |
| 6    | Annual Income: 35k 50k               |
| 7    | Annual Income: 25k 35k               |
| 8    | Annual Income: 10k 25k               |
| 9    | Annual Income: less 10k              |
| NA   | PMI: Skip / Prefer Not To Answer     |

---

### Education — concept 1585940 — 8 levels

| Code | AoU answer string                    |
|-----:|--------------------------------------|
| 1    | Highest Grade: Advanced Degree       |
| 2    | Highest Grade: College Graduate      |
| 3    | Highest Grade: College One to Three  |
| 4    | Highest Grade: Twelve Or GED         |
| 5    | Highest Grade: Nine Through Eleven   |
| 6    | Highest Grade: Five Through Eight    |
| 7    | Highest Grade: One Through Four      |
| 8    | Highest Grade: Never Attended        |
| NA   | PMI: Skip / Prefer Not To Answer     |

---

### Housing — concept 1585370 — 3 levels

| Code | AoU answer string                         |
|-----:|-------------------------------------------|
| 1    | Current Home Own: Own                     |
| 2    | Current Home Own: Rent                    |
| 3    | Current Home Own: Other Arrangement       |
| NA   | PMI: Skip / Prefer Not To Answer / Dont Know |

---

### Employment — concept 1585952 — 4 levels

Participants may select multiple employment categories; the **most deprived
value per person** is retained.

| Code | AoU answer string(s)                                                              |
|-----:|-----------------------------------------------------------------------------------|
| 1    | Employment Status: Employed For Wages; Employment Status: Retired                 |
| 2    | Employment Status: Self Employed                                                  |
| 3    | Employment Status: Out Of Work Less Than One                                      |
| 4    | Employment Status: Out Of Work One Or More; Employment Status: Unable To Work; Employment Status: Student |
| NA   | Employment Status: Homemaker; PMI: Skip / Prefer Not To Answer                    |

---

### Insurance — concept 1585386 — 2 levels

| Code | AoU answer string                         |
|-----:|-------------------------------------------|
| 1    | Health Insurance: Yes                     |
| 2    | Health Insurance: No                      |
| NA   | PMI: Skip / Prefer Not To Answer / Dont Know |

---

PC1 explains ~61% of variance across the five domains (polychoric
correlation matrix). After min-max reversal (so higher = more deprived),
PC1 correlates *r* ≈ 0.29 with zip-code area deprivation index — modest
but expected given individual- vs. area-level measurement mismatch.

## Notebooks

```
notebooks/
├── 01_sed_data_pull.py            # BQ pulls (Python / AoU Jupyter)
├── 02_sed_encoding_correlation.R  # Ordinal encoding, Spearman heatmaps
├── 03_sed_mice_imputation.R       # MICE (polr / logreg) on full cohort
└── 04_sed_pca_validation.R        # Polychoric PCA, min-max scaling,
                                   # validation vs. zip-code ADI
```

## Environment

All notebooks run inside an **AoU Controlled Tier** workspace on Terra.
The following environment variables must be set by the workspace:

```
WORKSPACE_CDR          BigQuery CDR dataset (e.g. fc-aou-cdr-prod.C2022Q4R9)
WORKSPACE_BUCKET       GCS bucket (gs://fc-secure-...)
OWNER_EMAIL            Terra user email
GOOGLE_PROJECT         Billing project
BIGQUERY_STORAGE_API_ENABLED  (Python only; presence enables BQ Storage API)
```

### Python dependencies (NB 01)
```
pandas
pandas-gbq
tqdm
```

### R dependencies (NB 02–04)
```r
install.packages(c("tidyverse", "bigrquery", "reshape2",
                   "mice", "psych", "readr", "lubridate"))
```

## Data flow

```
BQ ds_survey  ──►  01_sed_data_pull.py
                        │
                        ▼
              02_sed_encoding_correlation.R
              (ordinal encoding → sdi_var_df_new_order_with_spearman.csv)
                        │
                        ▼
              03_sed_mice_imputation.R
              (MICE → imputed_ds{1..5}_trial2.tsv)
                        │
                        ▼
              04_sed_pca_validation.R
              (polychoric PCA → individual_ses_pcs_normalised.csv)
```

All intermediate files are written to `$WORKSPACE_BUCKET/data/`.

## Key results

- Pre-imputation Spearman correlations range 0.14 (Employment–Insurance)
  to 0.60 (Income–Education), consistent with published area-level SED
  literature.
- Polychoric correlations are uniformly higher (0.31–0.68), as expected
  for ordinal data with bounded ranges.
- PC1 loading is approximately equal across all five domains (~0.43–0.51),
  supporting a single latent deprivation factor.

## Citation

If you use this pipeline, please cite:

> Gupta S, Lam V, Jordan IK, Mariño-Ramírez L. *A composite socioeconomic deprivation index from All of Us survey data: associations with health outcomes and disparities.* medRxiv 2024.10.04.24314904. PMID: 39802760. doi: [10.1101/2024.10.04.24314904](https://doi.org/10.1101/2024.10.04.24314904)

## Notes

- NB 01 also exists as an R version (NB 02 handles the same pull via
  `bq_table_save` + GCS export, which is the recommended path for large
  tables in AoU R workspaces).
- Employment allows multiple responses per person; the most deprived
  category per person is retained.
- Housing `PMI: Skip` is treated as NA (not coded as a deprivation level)
  in the final encoding.
