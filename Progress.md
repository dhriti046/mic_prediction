# DBAASP MIC Prediction Project — Detailed Progress Report

**Notebook:** `drive_linked_MIC_prediction_.ipynb`  
**Status:** The notebook contains the data-processing, feature, baseline, strain-aware modeling, evaluation, and inference workflows. The 30%-identity clustering cell has run in the current Colab session. Some newer cells—especially the ProtBERT baseline and Phase 4 experiments—still need to run successfully before their results can be reported. The report distinguishes implementation from verified execution.

> **Dataset scope:** The workspace preprocessing report and the model-ready notebook data describe different filtering stages. Counts below are labelled by source/scope; they should not be combined as one dataset total. The supplied preprocessing draft and the current cleaning summary also differ on sequence-length maximum, so that value should be reconciled before publication.

## 1. Dataset Collection and Preparation

Activity records were collected from DBAASP and organized by peptide, target, activity measurement, experimental metadata, and source information. The notebook includes a paginated peptide-list crawler, detail retrieval with retries, progress tracking, and saved activity rows. The preprocessing table preserves fields such as peptide ID, sequence, target species/strain, measurement ID, activity type/value, concentration, unit, medium, pH, inoculum, salt/ionic conditions, and reference where available.

The workspace cleaning summary reports **197,338 activity rows**, covering **24,919 unique peptides**. These are preprocessing-input counts, not the smaller model-ready counts.

## 2. MIC Data Cleaning

- Selected records with MIC activity for the MIC analysis while retaining non-MIC rows in the audit source.
- Parsed exact values, ranges, and censored operators separately; missing and unparsed values are tracked.
- Preserved original concentration/unit fields and bounds for traceability.
- The workspace summary reports **142,169 MIC rows**, **3,745 MIC values not parsed**, **37 unknown or unparsed MIC units**, and **113,145 rows with a µM conversion**.
- The notebook normalization keeps censored values as bounds rather than treating them as exact measurements. For ranges, both bounds are retained; a geometric-mean point estimate is available for exploratory modeling.

Mass-based conversions depend on a reliable molecular-weight estimate. The pipeline flags cases where that estimate is not appropriate, including modified or multimeric peptides.

## 3. Peptide Sequence Filtering

Missing and unusable sequences are identified, and a canonical-sequence subset is constructed from sequences containing only the 20 standard amino acids. The supplied preprocessing draft reports the filtered subset as **83,369 MIC records**, **11,847 unique sequences**, and **13,596 peptide IDs**.

The Phase 2 notebook uses a narrower exact-MIC modeling table. Its loader comments report **24,660 records** and **5,681 unique sequences** before feature-quality filtering. A further **33 single-residue records** were removed because DPC is undefined for one-residue sequences, leaving **24,627 model-ready records** according to the reported split sizes.

## 4. MIC Unit Conversion and pMIC Calculation

Supported molar units are standardized to micromolar concentration. Mass-based units are converted only when the peptide molecular weight can be estimated appropriately. Original values and units remain available alongside normalized values and conversion status.

For an MIC expressed in µM, the notebook calculates:

`pMIC = 6 − log10(MIC in µM)`

Point estimates are used for exact measurements. Censored values remain explicitly censored and are not silently converted into exact pMIC targets.

## 5. Sequence Length Analysis

Sequence length and distribution summaries are generated from peptide sequence metadata. The workspace cleaning summary reports a minimum length of **1** and maximum of **190** amino acids. The supplied filtered preprocessing draft reports a median of **16** amino acids and maximum of **140**. These values appear to refer to different source/filtering stages; recompute them from one named final subset before using them in a final report.

## 6. ESKAPEE Bacteria Standardization

Target strings are standardized to identify the seven ESKAPEE groups:

1. *Enterococcus faecium*
2. *Staphylococcus aureus*
3. *Klebsiella pneumoniae*
4. *Acinetobacter baumannii*
5. *Pseudomonas aeruginosa*
6. *Enterobacter* species
7. *Escherichia coli*

The notebook uses case-insensitive matching, handles common abbreviations such as “E. coli,” ignores strain suffixes for species-level group assignment, and audits unmatched target names. This supports ESKAPEE coverage analyses while preserving the original target label.

## 7. Target Annotation and Heuristic Extraction

The target annotation workflow extracts standardized species, strain text, Gram-positive/Gram-negative group, and candidate target level from raw target strings. Original labels are retained so heuristic annotations can be checked. Unmatched organisms outside the seven ESKAPEE groups can be reviewed rather than silently mapped to a pathogen group.

## 8. Sequence Clustering and Mapping

For the standard benchmark, CD-HIT clusters peptide sequences at **80% identity**. Short sequences that cannot be clustered by the usual protein word-length settings are assigned singleton IDs so each sequence remains represented in the split mapping.

The latest executed audit in the Colab session reports **2,328 train clusters**, **499 validation clusters**, and **499 test clusters**, with **zero cluster overlap** for all three pairwise comparisons. The equal validation/test counts reflect the requested split fractions; they are different cluster IDs. This latest total of 3,326 clusters differs from an earlier notebook summary that reported 2,409 clusters, so that summary should be refreshed from the exact split artifact before publication.

A separate **30% identity** benchmark was added using MMseqs2 because the CD-HIT protein mode used here does not accept a 30% cutoff. MMseqs2 temporary files are written to Colab local storage; result files are saved to Drive. The latest session advanced past the clustering cell and displayed **1,516 unique test sequences**. The model fits and final low-homology metrics still need review.

## 9. Leakage-Free Train/Validation/Test Splitting

The standard split assigns whole 80%-identity clusters to train, validation, or test. The audit checks train/validation, train/test, and validation/test cluster intersections and raises an error if any overlap is found.

The latest displayed record counts after the single-residue DPC filter are **17,976 train**, **3,271 validation**, and **3,380 test** records. The audit reports zero shared clusters across partitions.

## 10. Classic Sequence Feature Extraction (AAC and DPC)

- **AAC:** 20 normalized amino-acid frequency features.
- **DPC:** 400 normalized adjacent amino-acid-pair frequency features.
- **Quality checks:** finite values, AAC row sums of 1, and DPC row sums of 1 for sequences with at least two residues.
- **Filtering:** 33 single-residue records were removed from the model-ready feature splits because they have no adjacent residue pairs.

The ESM-2 merge cell was also updated after Colab emitted repeated DataFrame fragmentation warnings. It now joins the 320 embedding columns in one operation per split instead of inserting each column separately.

## 11. Physicochemical Property Extraction

Biopython ProtParam features include molecular weight, aromaticity, instability index, isoelectric point, GRAVY, aliphatic index, charge at pH 7, and sequence length. These descriptors are combined with sequence composition and ESM-2 features for model comparison.

## 12. Pretrained Protein Language Model Embeddings (ESM-2)

The notebook loads `esm2_t6_8M_UR50D`, mean-pools residue representations, and excludes special boundary tokens. Each peptide is represented by **320 embedding values**.

The embedding merge initially found 1,473 sequences missing from an existing partial `.npz` file. The extraction cell has been changed to resume from saved embeddings, compute missing sequences, and checkpoint each batch. The merge cell now verifies coverage and attaches all 320 dimensions in one operation. Run the updated extraction and merge cells to ensure the Drive file is complete and outputs reflect the latest code.

## 13. Data Leakage Remediation and Feature Hygiene

A leakage investigation identified measurement/outcome-related fields that could create implausibly strong predictions. The final numeric feature selection excludes MIC-derived or outcome-related fields and retains AAC, DPC, ESM-2, and physicochemical descriptors (**748 features** in the documented configuration). Clean target identity is added only in target-conditioned models.

The notebook includes feature correlation checks, target ablation, a leakage-free CatBoost fit, and feature ablation comparisons. Earlier exploratory metrics should not be interpreted as final results if they came from a feature set later found to contain leakage.

## 14. Modeling, Hyperparameter Tuning, and Model Comparison

Classical baselines include Random Forest and XGBoost across AAC, DPC, and combined feature configurations. The notebook also includes physicochemical/target-aware XGBoost, CatBoost hyperparameter comparisons, a full-feature XGBoost model, and a regression head trained on frozen ESM-2 embeddings.

The visible standard test comparison in the current Colab session reports:

| Model | MAE | RMSE | R² | Pearson r | Spearman ρ | Within 2× | Within 4× |
|---|---:|---:|---:|---:|---:|---:|---:|
| Strain-aware dual-branch NN | 0.4476 | 0.5758 | 0.2670 | 0.5254 | 0.5042 | 40.84% | 73.09% |
| Strain-aware CatBoost | 0.4444 | 0.5630 | 0.2992 | 0.5507 | 0.5272 | 41.08% | 72.50% |

These are the values visible in the supplied notebook screenshot; regenerate and retain the outputs after running the current notebook version.

### Phase 3 — Strain-Aware Model Development

- Implemented a dual-branch neural model combining the ESM-2 peptide vector and a learned target embedding.
- Implemented a strain-aware CatBoost model with a clean target label as a categorical feature.
- Added a 30%-identity MMseqs2 cluster-disjoint benchmark comparing peptide-only and target-conditioned CatBoost on the same held-out records. Clustering ran in the latest session; review the split and model metrics after the cell completes.
- Added a ProtBERT fine-tuning baseline for *E. coli* and *S. aureus* on the same peptide-cluster split, with matched comparisons against peptide-only and strain-aware CatBoost.
- **Pending:** The current Colab Transformers setup failed to load the legacy ProtBERT config through `AutoTokenizer`/`AutoModel`. The notebook now uses explicit `BertTokenizer` and `BertModel` classes; rerun cell 3.4 and confirm weights/tokenizer load and training completes.

### Phase 4 — Evaluation, Error Analysis, and MIC Resource Generation

- Existing analyses include per-pathogen performance, parity/calibration diagnostics, residual plots, and the `predict_mic()` inference function.
- Added a held-out-target experiment that excludes held-out target labels from model fitting and removes peptide-sequence overlap from training; it reports common and data-scarce target performance separately.
- Added error stratification by pMIC range, peptide length, target, and available assay-condition/uncertainty columns. Fields are reported as unavailable when they do not vary in the exact-MIC test set.
- Added a Drive bundle generator for normalized records, predictions, benchmark tables, and model artifacts. It creates a local bundle and does not publish data; verify DBAASP data-use terms before sharing.
- **Pending:** Run the Phase 4 experiments, inspect their outputs, regenerate the final comparison tables, and verify the release bundle contents.

## Remaining checks before reporting the project as complete

1. Rerun the updated ESM-2 extraction/merge cells and confirm there are no missing embeddings.
2. Rerun Phase 3.3 with MMseqs2 and verify the 30% cluster split and saved assignments.
3. Rerun Phase 3.4 after the explicit `BertTokenizer`/`BertModel` change; confirm the model loads and both target-specific runs finish.
4. Run the held-out-target and error-stratification cells in Phase 4.
5. Run the resource-bundle and final comparison cells; keep results for the standard, low-homology, held-out-target, and ProtBERT-matched test populations separate.
6. Reconcile dataset totals, sequence-length statistics, and cluster counts in the notebook summary with the final artifacts actually used.
7. Confirm whether the project intends to fine-tune ESM-2 weights or train a regression head over frozen ESM-2 embeddings. The current MLP head uses frozen embeddings; align the project statement with the chosen method.
