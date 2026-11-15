# MIC Prediction Using Antimicrobial Peptides

## 1. Project Overview

The goal of this project is to predict the Minimum Inhibitory Concentration (MIC) of antimicrobial peptides against different bacterial species and strains using machine learning and protein language models.

We collect experimental antimicrobial activity data from the DBAASP database, process and normalize measurements, and prepare the dataset for strain-aware MIC prediction.

The primary objective is to predict peptide activity against specific bacterial targets rather than treating antimicrobial activity as independent of the target organism.

## 2. What Is MIC?

MIC stands for Minimum Inhibitory Concentration. It is the minimum concentration of an antimicrobial agent required to inhibit visible microbial growth under specified experimental conditions.

- Lower MIC generally indicates stronger antimicrobial activity under comparable conditions.
- MIC values may be reported in different units.
- Experimental conditions, measurement ranges, and target organisms must be considered when comparing measurements.

## 3. Data Source

We use the Database of Antimicrobial Activity and Structure of Peptides (DBAASP).

- Website: https://dbaasp.org/
- Peptide listing API: https://dbaasp.org/peptides
- Individual peptide details: `https://dbaasp.org/peptides/{peptide_id}`

The listing API supports pagination using `limit` and `offset` parameters.

During the initial API test, DBAASP reported 25,542 peptide records. This was the count observed during that test, not the number of activity measurements or a verified current database total.

## 4. Project Workflow

<img width="1660" height="4520" alt="mermaid-diagram (1)" src="https://github.com/user-attachments/assets/3cd5bf79-ef07-4f77-9bef-191304a795ec" />


## Architecture explanation
### Data collection: Collect peptide sequences, MIC values and bacterial targets.
### Preprocessing: Clean data and standardise MIC into pMIC.
### Target processing: Identify bacterial species and strains.
### Sequence clustering: Group similar peptides to prevent data leakage.
### Data splitting: Create training, validation and testing sets.
### Feature extraction:	Generate AAC, DPC, physicochemical and ESM-2 features.
### Feature integration:	Combine 748 sequence-derived features.
### Leakage audit:	Remove columns that reveal the target value.
### Model development: Train Random Forest, XGBoost and CatBoost.
### Optimisation:	Tune CatBoost and compare feature combinations.
### Prediction:	Predict antimicrobial activity using pMIC.
### Evaluation: Measure performance using MAE, RMSE and R².
