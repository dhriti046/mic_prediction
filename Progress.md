# MIC Prediction Project Progress

**Notebook tracked here:** `drive_linked_MIC_prediction_.ipynb`  
**Status:** Main notebook contains the workflow and modeling code described below. Its saved outputs and execution counts are from earlier runs and have not been confirmed by a clean end-to-end run of the current code.

## Completed in the main notebook

### Phase 1 — DBAASP data collection and dataset construction

- Added a paginated DBAASP peptide-list crawler that saves pages to Google Drive, and an activity-detail extractor with retries, progress tracking, and CSV output.
- Reconstructed the MIC activity table while preserving peptide, target, measurement, and API metadata.
- Added sequence validation and MIC parsing for exact, interval, censored, missing, and invalid values. The normalization pipeline converts supported molar units and conditionally converts mass-based units when peptide molecular weight can be estimated.
- Computes `pMIC = 6 − log10(MIC in µM)` for valid point estimates; interval and censored measurements are retained with relation/bound fields rather than treated as exact outcomes.
- Added ESKAPEE grouping and summary analyses, plus target annotations including `target_original`, `species_name`, `strain_name`, `bacteria_type`, and `target_level`.
- Exports exact-MIC, ESKAPEE, target-coverage, and model-ready datasets to Google Drive.
- Added a review checkpoint that pauses before clustering and modeling until the curated dataset is reviewed.

### Phase 2 — Sequence features and prediction baselines

- Builds an 80%-identity CD-HIT cluster map and assigns clusters before creating cluster-grouped train, validation, and test splits.
- Extracts 20 AAC and 400 DPC features, physicochemical descriptors, and 320-dimensional mean-pooled ESM-2 embeddings.
- Compares Random Forest, XGBoost, and CatBoost models across feature configurations; includes hyperparameter search, feature ablation, and target-aware models.
- Adds a leakage audit and a numeric feature set containing AAC, DPC, ESM-2, and physicochemical descriptors.
- Includes expanded metrics: MAE, RMSE, R², Pearson r, Spearman ρ, and twofold/fourfold MIC error rates, plus an ESM-2 regression head.

### Phase 3 — Strain-aware modeling

- Implements a dual-branch neural predictor combining ESM-2 peptide embeddings with a learned bacterial-target embedding.
- Adds a strain-aware CatBoost model using the target as a categorical feature.
- Audits train/test cluster separation at the configured 80% CD-HIT threshold.

### Phase 4 — Evaluation and inference

- Produces pathogen-level test metrics, parity/calibration/error plots, and residual analyses.
- Provides a `predict_mic()` inference function that computes AAC, DPC, physicochemical features, and ESM-2 embeddings using the model's feature definitions, then converts predicted pMIC to µM.
- Labels the twofold output as a fold-error range rather than a statistical confidence interval.
