# aou-sed-gwas

Individual-level Social and Economic Determinants (SED) index construction
from the **All of Us** Controlled Tier Dataset v7, for use as covariates in
downstream GWAS and PRS analyses.

## Overview

Constructs a polychoric-PCA-based individual deprivation index from five
AoU survey domains:

| Variable   | Concept ID | Levels | Encoding (1 = least deprived) |
|------------|-----------|--------|-------------------------------|
| Income     | 1585375   | 9      | 1 = >$200k … 9 = <$10k       |
| Education  | 1585940   | 8      | 1 = Advanced Degree … 8 = Never Attended |
| Housing    | 1585370   | 3      | 1 = Own, 2 = Rent, 3 = Other |
| Employment | 1585952   | 4      | 1 = Employed/Retired … 4 = Unable to Work |
| Insurance  | 1585386   | 2      | 1 = Yes, 2 = No               |

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
