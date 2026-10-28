```markdown
# Neuro-Symbolic Network Anomaly Detection on UNSW-NB15

**Integrating Autoencoder-Based Detection with Knowledge-Based Rules**

University of Greenwich · COMP1818 Artificial Intelligence Applications  
Author: Viveka Sattlapally (`vs2806m@gre.ac.uk`)

This repository contains the code, experimental pipeline, and supporting artefacts for a controlled evaluation of whether a small set of heuristic symbolic rules improves unsupervised network anomaly detection on the UNSW-NB15 dataset.

The study compares classical unsupervised detectors, a deep autoencoder, a Kitsune-inspired random-subspace ensemble, three percentile-based rules, and a validation-tuned hybrid fusion. All primary claims are supported by matched ablations (rules-zeroed and label-tuned threshold controls) and multi-seed analysis.

> **Key result (single seed 42):** Hybrid MCC = 0.6374, rules-zeroed ablation MCC = 0.6387.  
> Across five seeds the paired difference is +0.0004 ± 0.0028 (95 % CI [−0.0031, +0.0039]), indistinguishable from zero.  
> The observed single-seed gain is a threshold-calibration effect, not evidence of additional semantic value from the rules.

---

## Table of Contents

- [Research Question](#research-question)
- [Key Findings](#key-findings)
- [Methodology Overview](#methodology-overview)
- [Primary Results](#primary-results)
- [Dataset & Data Quality](#dataset--data-quality)
- [Repository Structure](#repository-structure)
- [Installation](#installation)
- [Dataset Setup](#dataset-setup)
- [Running the Pipeline](#running-the-pipeline)
- [Reproducibility](#reproducibility)
- [Limitations](#limitations)
- [References](#references)
- [License](#license)

---

## Research Question

Do manually specified percentile-based symbolic rules improve the detection performance of an autoencoder-based network intrusion detection system on UNSW-NB15, and how robust are any observed gains to threshold selection, duplicate records, and random-seed variation?

---

## Key Findings

| Finding | Detail |
|---------|--------|
| Baseline autoencoder | Test MCC 0.6259 (FPR 0.0915) |
| Hybrid (rules + AE) | Test MCC 0.6374 (FPR 0.0645) |
| Rules-zeroed ablation | Test MCC 0.6387 — algebraically equivalent to the AE at threshold 1.667 |
| Multi-seed paired difference | Hybrid − zeroed = +0.0004 ± 0.0028 (5 seeds) |
| Rule contribution | Rules change only 105 of 82 332 test decisions; 37 of those 105 are attacks (35 %) |
| Symbolic rules alone | Attack recall 14.65 %, FPR 2.74 % |
| Random-subspace ensemble | Unstable (MCC 0.469 ± 0.107) |
| Unseen-category indicators | No measurable effect |
| Deduplication | Removing within-test model-visible duplicates lowers unsupervised MCC by 0.14–0.32 |
| Supervised comparator | Random Forest MCC 0.7506 at 27.4 % FPR (reference only) |

The hybrid does **not** demonstrate a robust advantage once threshold confounds are isolated.

---

## Methodology Overview

1. **Data**  
   Official UNSW-NB15 train/test split (175 341 / 82 332 records).  
   Columns dropped: identifiers, attack category, TCP sequence numbers, and TTL-related features (precaution against testbed topology artefacts).  
   Heavy-tailed volume features log-transformed; scaler and one-hot encoder fitted on normal training rows only.  
   Three binary indicators flag categorical values unseen in normal training traffic (final dimension = 63).

2. **Internal splits**  
   Group-aware `GroupShuffleSplit` on model-visible feature hashes → train / val_es / val_rules / val_fusion (no model-visible vector crosses internal subsets).

3. **Models**  
   - Isolation Forest & One-Class SVM (classical baselines)  
   - Deep autoencoder (63→32→16→32→63) trained on normal traffic only; score = reconstruction MSE / 95th-percentile normal validation MSE  
   - Random-subspace ensemble of eight small autoencoders + output autoencoder (Kitsune-inspired; random partition, **not** correlation clustering)  
   - Three percentile rules (flood-like, asymmetric-outbound, scan-like) selected under a 2 % FPR cap on val_rules  
   - Hybrid fusion  
     \[
     S_{\text{Hybrid}} = \alpha S_{\text{ML}} + (1-\alpha)S_{\text{Sym}} + \gamma(S_{\text{ML}}\cdot S_{\text{Sym}})
     \]
     with \(\alpha=0.6\), \(\gamma=0.1\), \(t=1.0\) (selected on val_fusion).  
     When no rule fires this is equivalent to \(S_{\text{ML}}>1.667\).

4. **Evaluation**  
   Held-out test metrics, multi-seed sensitivity (seeds 42, 7, 123, 2024, 99), attack-category recall, deduplication sensitivity, and rules-zeroed / label-tuned controls.

---

## Primary Results

Test set (82 332 records, 55.1 % attack prevalence)

| Model                              | MCC    | FPR    |
|------------------------------------|--------|--------|
| Isolation Forest                   | 0.2907 | 0.0702 |
| One-Class SVM                      | 0.6125 | 0.0648 |
| Symbolic rules only                | 0.2039 | 0.0274 |
| Deep autoencoder                   | 0.6259 | 0.0915 |
| AE label-tuned control             | 0.6395 | 0.0672 |
| Random-subspace ensemble           | 0.5568 | 0.0788 |
| Logical OR                         | 0.6077 | 0.1161 |
| Logical AND                        | 0.2515 | 0.0028 |
| Rules-zeroed ablation              | 0.6387 | 0.0627 |
| **Hybrid**                         | **0.6374** | **0.0645** |
| Supervised Random Forest (reference) | 0.7506 | 0.2743 |

---

## Dataset & Data Quality

- **Source**: Official UNSW-NB15 training and testing CSV files  
  (Kaggle mirror: https://www.kaggle.com/datasets/mrwellsdavid/unsw-nb15)  
- Model-visible internal duplicates: 74 303 (train), 28 386 (test)  
- Model-visible cross-split overlaps: 1 305  
- Primary evaluation is **not** leakage-free with respect to the official split; internal group-aware splits only protect the validation subsets.

---

## Repository Structure

```text
COMP1818-NIDS-Project/
├── nids_pipeline.py          # Full experimental pipeline
├── requirements.txt
├── README.md
├── LICENSE
├── references_AI.bib
└── artifacts/                # Generated CSVs, model weights, configs
    ├── final_experimental_results.csv
    ├── multi_seed_sensitivity.csv
    ├── experiment_config.json
    ├── autoencoder_state_dict.pt
    └── ...
```

---

## Installation

```bash
git clone https://github.com/vs2806m/COMP1818-NIDS-Project.git
cd COMP1818-NIDS-Project

python3 -m venv .venv
source .venv/bin/activate          # Windows: .\.venv\Scripts\Activate.ps1

python -m pip install --upgrade pip
pip install -r requirements.txt
```

**Requirements**: Python ≥ 3.10, PyTorch, scikit-learn, pandas, NumPy, joblib.

---

## Dataset Setup

1. Download the official `UNSW_NB15_training-set.csv` and `UNSW_NB15_testing-set.csv`.
2. Place them in the project root **or** set the environment variable:
   ```bash
   export DATA_DIR="/path/to/csv/directory"
   ```
3. Do **not** commit the raw dataset files to the repository.

---

## Running the Pipeline

```bash
python nids_pipeline.py
```

The script performs data audit, group-aware splitting, model training, rule selection, fusion grid search, ablations, multi-seed sensitivity, and writes all artefacts to the working directory / `artifacts/`.

---

## Reproducibility

- Global seed control (Python, NumPy, PyTorch, CuDNN deterministic mode).  
- Primary seed = 42; sensitivity seeds = [42, 7, 123, 2024, 99].  
- Preprocessor fitted exclusively on normal training rows.  
- Internal splits use `GroupShuffleSplit` on SHA-256 hashes of model-visible feature vectors.  
- All thresholds, fusion parameters and rule thresholds are recorded in `experiment_config.json`.  
- Model weights and preprocessor objects are saved for exact reload.

---

## Limitations

1. Rules are simple percentile heuristics, not an expert knowledge base.  
2. Apparent hybrid gains are threshold-calibration effects.  
3. Cross-split model-visible overlaps exist in the official UNSW-NB15 files.  
4. Attack-category performance is highly uneven (Shellcode recall = 0 %).  
5. Only five random seeds were evaluated.  
6. Results are specific to the UNSW-NB15 flow-level feature set and do not generalise to live traffic or packet-level features.  
7. The ensemble is Kitsune-**inspired** (random partition); it is not a faithful reproduction of KitNET’s correlation-based mapper.

---

## References

1. Moustafa & Slay (2015). UNSW-NB15: A comprehensive data set for network intrusion detection systems. *MilCIS*.  
2. Moustafa & Slay (2016). The evaluation of Network Anomaly Detection Systems. *Information Security Journal*.  
3. Mirsky et al. (2018). Kitsune: An Ensemble of Autoencoders for Online Network Intrusion Detection. *NDSS*.  
4. Liu, Ting & Zhou (2008). Isolation Forest. *ICDM*.  
5. Schölkopf et al. (2001). Estimating the Support of a High-Dimensional Distribution. *Neural Computation*.  
6. Sommer & Paxson (2010). Outside the Closed World. *IEEE S&P*.  
7. d’Avila Garcez et al. (2019). Neural-symbolic computing. *Journal of Applied Logics*.  
8. Arp et al. (2022). Dos and Don’ts of Machine Learning in Computer Security. *USENIX Security*.

Full BibTeX entries are provided in `references_AI.bib`.

---

## License

This repository’s original code is released under the [MIT License](LICENSE).

The MIT License covers only the software in this repository. The UNSW-NB15 dataset is subject to the terms set by its creators (Australian Centre for Cyber Security) and is intended for non-commercial academic research. Third-party libraries remain under their respective licences.
```

This version is concise, accurate to your experimental results, professionally structured, and ready to paste into the repository root as `README.md`.
