# DBAASP MIC Dataset Preprocessing

## 1. Dataset Collection and Preparation

- **Source:** Antimicrobial peptide activity data were retrieved from the **DBAASP** database via its public REST API.
- **Content:** The raw dump contains measurements for many activity types (MIC, IC₅₀, etc.). For this project we focused on **MIC (Minimum Inhibitory Concentration)** records.
- **Extracted Fields:**
  - `peptide_id` – Unique identifier for each peptide in DBAASP.
  - `target_species` – Bacterial species/strain the peptide was tested against.
  - `activity_type` – (filtered to `"MIC"`).
  - `concentration` – Numerical value(s) of the activity.
  - `unit` – Original unit string (e.g., µM, mM, mg/ml, etc.).
  - `sequence_raw` – Original peptide sequence (as stored in DBAASP).
  - `complexity_api` – Sequence complexity metric provided by DBAASP.
  - `sequence_length_api` – Length as reported by DBAASP.

The extracted table was saved as `raw/mic_raw.csv`.

---

## 2. MIC Data Cleaning

1. **Select MIC only** – Filtered rows where `activity_type == "MIC"`.
2. **Column reduction** – Kept only the columns required for downstream work:
   - `peptide_id`
   - `target_species`
   - `activity_measure` (kept for completeness)
   - `concentration`
   - `unit`
   - `sequence_raw`
   - `complexity_api`
   - `sequence_length_api`
3. **Missing/invalid concentrations** – Rows with missing or non‑numeric concentration values were dropped.
4. **Range handling** – For entries expressed as a range (e.g., `5‑10 µM` or `5 to 10 µM`) we computed the **geometric mean**: \(\sqrt{low \times high}\) and stored the result as a single numeric value.
5. **Add sequence length & complexity** – Pulled `sequence_length_api` and `complexity_api` from the original metadata to augment the table.
6. **Bound removal** – Records where the concentration column contained only an upper or lower bound (e.g., `<0.5 µM`, `>100 µM`) were removed because they do not provide a precise numeric value for regression.

---

## 3. Peptide Sequence Filtering

- **Missing sequences** – Rows where `sequence_raw` was empty/NaN were discarded.
- **Canonical filtering** – Created a **canonical subset** consisting exclusively of sequences composed of the 20 standard amino acids (`ACDEFGHIKLMNPQRSTVWY`). Non‑canonical symbols (e.g., `U`, `O`, `B`, `Z`, `X`, lower‑case letters for D‑amino acids) were removed.
- **Statistics after filtering:**
  - **Total MIC records:** 83,369
  - **Unique peptide sequences:** 11,847
  - **Unique peptide IDs:** 13,596

---

## 4. MIC Unit Conversion and pMIC Calculation

### 4.1 Standardising Units
- Normalised unit strings to a lower‑case, whitespace‑free form.
- Mapped molar units (`M`, `mM`, `µM`, `nM`, `pM`) to a conversion factor to **micromolar (µM)**.
- For mass‑based units (`µg/ml`, `mg/ml`, `ng/ml`) we estimated the peptide molecular weight (see section 11) and applied the conversion:
  \[\text{µM} = \frac{\text{mass concentration} \times 10^3}{\text{MW (Da)}}\]

### 4.2 pMIC Calculation
- After conversion to µM, we transformed the concentration to a **log‑scaled pMIC** using the convention adopted in the DBAASP MIC dataset PDF:
  \[\text{pMIC} = 6 - \log_{10}(\text{MIC}_{\mu M})\]
- Both the original concentration (`concentration_original`) and the transformed `pMIC` columns were retained in the final table.

---

## 5. Sequence Length Analysis

- Utilised the `sequenceLength` column (derived from the canonical filter) to compute descriptive statistics:
  - **Minimum length:** 1 aa
  - **Maximum length:** 140 aa
  - **Median length:** 16 aa
- The distribution is heavily skewed toward short peptides (majority ≤ 20 aa), with a long‑tail of longer sequences.

---

## 6. ESKAPEE Bacteria Standardization

- **Goal:** Collapse the many textual variations of bacterial names into the seven **ESKAPEE** pathogen groups.
- **Target groups:**
  1. *Enterococcus faecium*
  2. *Staphylococcus aureus*
  3. *Klebsiella pneumoniae*
  4. *Acinetobacter baumannii*
  5. *Pseudomonas aeruginosa*
  6. *Enterobacter* spp.
  7. *Escherichia coli*
- **Regex pipeline:**
  - Lower‑case the raw target string.
  - Remove strain/serotype information after the first space.
  - Match using flexible patterns (e.g., `e.?\s*col(i)?` → `escherichia coli`).
- Unmatched names were assigned `None` and inspected manually; no additional ESKAPEE entries were missed.

---

## 7. Target Annotation & Heuristic Extraction

- Developed a **structured target schema** with the following fields:
  - `species_name` – Canonical species (lower‑case).
  - `strain_name` – Strain identifier if available (e.g., `ATCC 25922`).
  - `bacteria_type` – `Gram‑positive` or `Gram‑negative` (derived from a lookup table).
  - `target_level` – `species` vs `strain` level based on presence of strain information.
- Implemented using a series of regular‑expression matches. Audited edge cases such as *Staphylococcus epidermidis* and *Enterococcus faecalis* (outside the core seven groups) and flagged them for optional downstream analysis.

---

## 8. CD‑HIT Sequence Clustering & Mapping

1. **Installation** – Ran `!apt-get install -y cd-hit` in the Colab environment.
2. **FASTA creation** – Generated a FASTA file containing all **canonical** sequences of length **≥ 5** residues from the ESKAPEE‑filtered subset.
3. **Clustering** – Executed `cd-hit -i input.fasta -o clusters.fasta -c 0.80 -aS 0.8 -aL 0.8` to obtain **80 % identity** clusters with global alignment coverage constraints.
4. **Mapping** – Parsed the `.clstr` file to assign each peptide a **stable cluster ID**. Short sequences (< 5 residues) that could not be clustered were given a **singleton ID**.
5. **Resulting columns:** `cluster_id` (stable cluster identifier) and `is_singleton` (bool).

---

## 9. Leakage‑Free Train‑Validation‑Test Splitting

- Used **`GroupShuffleSplit`** from scikit‑learn with `groups=cluster_id` to enforce that **no cluster appears in more than one split**.
- Performed a nested split: **train (72.9 %) → validation (13.2 %) → test (13.8 %)**.
- Verified **zero overlap** of both `cluster_id` and raw peptide sequences across the three partitions.

---

## 10. Classic Sequence Feature Extraction (AAC & DPC)

- **AAC (Amino‑Acid Composition):** 20‑dimensional vector of normalized frequencies for each standard residue.
- **DPC (Dipeptide Composition):** 400‑dimensional vector of normalized frequencies for each ordered di‑peptide (AA, AC, …, YY).
- **Filtering:** Removed **33** single‑residue records (all lysine monomers) because di‑peptide frequencies are undefined for length 1.
- **Validation:** Confirmed that each AAC/DPC vector sums to 1 (within floating‑point tolerance).

---

## 11. Physicochemical Property Extraction

- Installed **Biopython** (`!pip install -q biopython`).
- Utilised `Bio.SeqUtils.ProtParam.ProteinAnalysis` to compute eight descriptors for every **unique** peptide sequence:
  1. **Molecular Weight** (Da)
  2. **Aromaticity** (fraction of aromatic residues)
  3. **Instability Index** (predicts stability in vitro)
  4. **Isoelectric Point (pI)**
  5. **GRAVY** – Grand Average of Hydropathy
  6. **Aliphatic Index**
  7. **Charge at pH 7.0**
  8. **Sequence Length** (redundant with earlier length column but kept for convenience)
- These features were merged back into the train/validation/test CSVs.

---

## 12. Pre‑trained Protein Language Model Embeddings (ESM‑2)

1. **Installation** – `!pip install -q fair-esm`.
2. **Model** – Loaded `esm2_t6_8M_UR50D` (320‑dimensional embeddings, GPU‑accelerated when available).
3. **Batch generation** – Processed unique sequences in batches of 16, applied **mean‑pooling** over residue embeddings (ignoring BOS/EOS tokens).
4. **Storage** – Saved the resulting matrix as `esm2_t6_8M_embeddings.npz` (atomic write to Google Drive). The file maps `sequence_id` → embedding vector.
5. **Integration** – Merged the embedding columns (`ESM_000` … `ESM_319`) into each of the three data splits.

---

## 13. Data Leakage Remediation & Feature Hygiene

- **Problem discovered:** Certain downstream target columns (e.g., `target_normalized`) showed **very high correlation** (> 0.9) with `pMIC_point`, indicating a leakage source.
- **Resolution:** Constructed a **leakage‑free feature set** consisting of:
  - AAC (20)
  - DPC (400)
  - ESM‑2 embeddings (320)
  - Physicochemical descriptors (8)
  - **Total:** 748 clean numeric features.
- Verified that none of the selected columns correlate > 0.3 with the target after removal of the leaked columns.

---

## 14. ML Modeling, Hyper‑parameter Tuning & Model Comparison

### Baseline Classical Models
- **Random Forest**, **XGBoost**, **CatBoost** trained on various feature configurations (AAC only, AAC + DPC, AAC + PhysChem, ESM‑2 only, Combined).
- Evaluated using **MAE**, **RMSE**, **R²**, **Pearson (r)**, and **Spearman (ρ)** on the validation split.

### CatBoost Specifics
- Utilised **GPU‑accelerated** CatBoost (`task_type='GPU'` when `torch.cuda.is_available()` is true).
- **RandomizedSearchCV** over:
  - `depth` ∈ {4,5,6,7,8}
  - `learning_rate` ∈ {0.01,0.03,0.05,0.07}
  - `l2_leaf_reg` ∈ {1,3,5,7,10}
  - `bagging_temperature` ∈ {0,0.5,1}
- **Best model** achieved **MAE ≈ 0.44**, **R² ≈ 0.35** on the validation set (exact numbers logged in `catboost_validation_metrics.csv`).

### Feature Ablation Study
- Trained models on the following configurations and recorded validation performance:
  1. **AAC only**
  2. **AAC + DPC**
  3. **AAC + PhysChem**
  4. **ESM‑2 only**
  5. **AAC + ESM‑2**
  6. **AAC + ESM‑2 + PhysChem** (full set)
- Results demonstrated a clear **performance boost** when adding ESM‑2 embeddings and physicochemical descriptors, confirming the value of language‑model representations.

---

## 15. Deliverables
- **Processed CSVs:** `phase2_train.csv`, `phase2_validation.csv`, `phase2_test.csv` (all features + target).
- **Embedding file:** `esm2_t6_8M_embeddings.npz`.
- **Trained model checkpoints:** CatBoost (`catboost_best.joblib`), XGBoost (`xgb_combined.joblib`), and the strain‑aware PyTorch model (`strain_aware_mic.pt`).

---