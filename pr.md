# PR: Refactor `latest_final_code.ipynb` into a clean, Drive-cached ablation pipeline

- Rebuilt the 185-cell notebook into 23 clean cells with numbered Markdown sections (1–10); removed debug cells, duplicate imports, and dead code.
- Mount Google Drive once and pin a single `PROJECT_DIR`; added a compute-only-if-missing cache convention used for every heavy artifact.
- GPU-aware throughout: `DEVICE = cuda if available else cpu`, with a fixed seed (`set_seed`) applied before every model.
- Load cached cluster-disjoint 75/15/10 split (`phase2_{train,validation,test}.csv`) instead of re-running DBAASP collection / CD-HIT clustering.
- ESM-2 peptide embeddings cached to Drive as `.npz`; computed only for missing sequences. Backbone is a single switch `ESM_BACKBONE` ("8M" → 320-d default, "150M" → 640-d) via an `ESM_REGISTRY`; each backbone caches to its own file so both are kept and A/B-able, and all downstream heads read `ESM_DIM` dynamically. Extraction is resume-safe: it checkpoints the `.npz` atomically every ~320 sequences, so a dropped kernel (common on long 150M runs) resumes from the last checkpoint instead of recomputing everything.
- Mapped each unique target species to a 2-sentence biological description (Gram stain, cell wall, resistance), encoded once with PubMedBERT (mean-pooled, `torch.no_grad`), cached as `species_pubmedbert_emb.pt`.
- Added categorical taxonomy features (`gram_stain`, `genus`, `family`) via one-hot encoding fit on train only; no GNNs.
- Implemented a single shared MLP head (Huber loss, same optimizer/schedule/seed/split) driving a 5-way ablation: Peptide-only, +Strain ID (learned embedding), +Species Description, +Taxonomy, +Shuffled-Description control.
- Added three separate (non-mixed) evaluations: standard clustered split, zero-shot held-out targets (with peptide-overlap removal), and few-shot sparse targets (5–20 train rows).
- Added long-tail analysis: per-target MAE/RMSE/Spearman ρ/Kendall τ bucketed Low (<20)/Medium (20–100)/High (>100), with one summary table and a grouped bar chart across all 5 models.
- Added FiLM (Feature-wise Linear Modulation) section: the PubMedBERT pathogen embedding generates γ/β that modulate the ESM-2 peptide embedding (`(1+γ)⊙h+β`), compared head-to-head against plain concatenation of the same species-description signal across all three protocols (`film_vs_concat.csv`).
- Added terminal-modification flags parsed from DBAASP `api_modifications_json` (C-terminal amidation, N-terminal acetylation), with graceful fallback if the column is absent.
- Added short-peptide biophysical descriptors (net charge at pH 7, hydrophobic moment, Boman index, length), z-scored on train stats and fused with ESM-2.
- Added resistance-aware strain text: resistance phenotypes (MRSA/VRE/carbapenem/ESBL/MDR/efflux/colistin) parsed from the strain label and appended to the PubMedBERT description, cached per strain as `strain_pubmedbert_emb.pt`.
- Added an Enriched model (ESM-2 + biophysics + mod flags + resistance-aware text) compared to peptide-only and species-description baselines across all three protocols (`enriched_results.csv`).
- Added the best model (Section 12): Strain-ID anchor (learned strain embedding) + enriched peptide stream (ESM-2 + biophysics + mod flags) with FiLM gating by the resistance-aware pathogen embedding; uses Peptide+StrainID (strongest few-shot baseline) as the comparison anchor instead of Peptide-only, across all three protocols and on the long tail (`best_model_results.csv`, `best_model_long_tail.csv`).
- Added MSE to the shared metrics (now N/MAE/MSE/RMSE/R²/Pearson/Spearman/Kendall) so every table, including few-shot with the Peptide+StrainID row, reports all of them.
- Added a final PathoMIC benchmark-comparison cell (arXiv 2608.26228v1): our ESM2-8M and best model vs PathoMIC's ESM2-8M and best (SP/GNN) on MSE/Spearman/Kendall for few-shot and zero-shot, with explicit caveats that protocol/data differ (`pathomic_comparison.csv`).
- All run outputs saved under `PROJECT_DIR/ablation_cache` (per-evaluation CSVs, long-tail tables, `long_tail_mae.png`).

## Architecture

Two **frozen** backbones (ESM-2 for peptides, PubMedBERT for pathogen text) feed
lightweight trainable heads (MLP / FiLM). All learning is in the heads; both
encoders are run once and cached to Drive.

### Data & caching layer
```
Google Drive (PROJECT_DIR = dbaasp_backup)
├── phase2_output/phase2_{train,validation,test}.csv   # cluster-disjoint 75/15/10 split (loaded)
└── ablation_cache/                                     # computed once, cached:
    ├── esm2_t6_8M_peptide_emb.npz        # 320-d ESM-2 peptide embeddings
    ├── species_pubmedbert_emb.pt         # 768-d species-description embeddings
    ├── strain_pubmedbert_emb.pt          # 768-d resistance-aware strain embeddings
    └── *.csv / long_tail_mae.png         # all result tables + chart
```
Every heavy artifact is check-exists → load, else compute → save. GPU-aware
(`DEVICE = cuda if available else cpu`); fixed seed before every model.

### Feature streams
- **Peptide side:** frozen ESM-2 (`esm2_t6_8M`, 320-d, mean-pooled) + biophysics
  (net charge pH 7, hydrophobic moment, Boman index, length) + terminal-mod flags
  (C-amidation, N-acetylation) + learned Strain-ID embedding.
- **Pathogen side:** frozen PubMedBERT (768-d) on a 2-sentence species description
  (resistance-aware variant appends strain phenotypes) + one-hot taxonomy
  (gram / genus / family). No GNNs.

### Models
1. `AblationMLP` — one shared head (2 hidden layers, LayerNorm/GELU/dropout,
   Huber loss); drives the 5-way ablation, only the concatenated input block changes.
2. `FiLMRegressor` — pathogen embedding generates γ/β modulating the peptide
   embedding: `h = (1 + γ) ⊙ h + β`.
3. `FiLMEnriched` (best) — Strain-ID embedding ⊕ enriched peptide stream,
   FiLM-gated by the resistance-aware pathogen embedding.

### Flow
```
load split
  → ESM-2 (cache) → PubMedBERT species + strain (cache) + taxonomy
  → 5-way ablation ──┬── standard split ──┬── long-tail (per-target, bucketed)
                     ├── zero-shot ───────┤
                     └── few-shot ────────┘
  → FiLM vs concat
  → best model (anchor = Peptide+StrainID)
  → PathoMIC benchmark comparison
```

### Study matrix
- 5-way ablation (same head/split/seed): Peptide-only · +Strain-ID · +Species-Desc
  · +Taxonomy · +Shuffled-Desc control.
- 3 separate evaluations: standard clustered · zero-shot (unseen strains,
  cluster-disjoint) · few-shot (5–20 train rows).
- Long-tail: per-target metrics bucketed Low/Medium/High + grouped bar chart.
- FiLM vs concat on the same pathogen signal.
- PathoMIC comparison (MSE / Spearman / Kendall, few-shot & zero-shot).
