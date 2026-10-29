<div align="center">

# Neuro-Symbolic Network Anomaly Detection on UNSW-NB15

### Do Percentile-Based Rules Add Value to an Autoencoder NIDS? A Controlled Test with a Kitsune-Inspired Ensemble

[![Python](https://img.shields.io/badge/Python-3.13-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![PyTorch](https://img.shields.io/badge/Framework-PyTorch-EE4C2C?logo=pytorch&logoColor=white)](https://pytorch.org/)
[![Dataset](https://img.shields.io/badge/Dataset-UNSW--NB15-20BEFF?logo=kaggle&logoColor=white)](https://www.kaggle.com/datasets/mrwellsdavid/unsw-nb15)
[![License](https://img.shields.io/badge/License-MIT-green)](https://github.com/vs2806m/COMP1818-NIDS-Project/blob/main/LICENSE)
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/vs2806m/COMP1818-NIDS-Project/blob/main/nids_pipeline.ipynb)

**University of Greenwich · COMP1818 Artificial Intelligence Applications**

</div>

---

## Overview

This repository contains the code, trained models and saved outputs for a
controlled evaluation of whether three simple, percentile-based rules improve
unsupervised network anomaly detection on the **UNSW-NB15** dataset.

A deep autoencoder is compared with classical unsupervised detectors, a
Kitsune-inspired random-subspace ensemble, the rules on their own, and a
validation-tuned fusion of the autoencoder and the rules. Two matched
controls (a rules-zeroed ablation and a label-tuned threshold), five random
seeds and a deduplicated-test analysis separate the effect of the rules from
threshold choice, duplicate records and training noise.

> **Main finding**
>
> The hybrid detector reached a test MCC of **0.6374**, against **0.6387** for
> the rules-zeroed control. Across five seeds the paired difference was
> **+0.0004 ± 0.0028** (95% CI [−0.0031, +0.0039]). Under this protocol the
> three rules add **no measurable detection benefit**: the hybrid's single-seed
> gain over the plain autoencoder is a threshold adjustment and does not
> replicate across seeds.

## Contents

1. [Quick start](#quick-start)
2. [Research question](#research-question)
3. [Key results](#key-results)
4. [Methodology](#methodology)
5. [Dataset and data quality](#dataset-and-data-quality)
6. [Repository structure](#repository-structure)
7. [Reproducibility](#reproducibility)
8. [Limitations](#limitations)
9. [References](#references)
10. [Licence and terms of use](#licence-and-terms-of-use)
11. [AI use declaration](#ai-use-declaration)

---

## Quick start

### 1. Install

Tested with **Python 3.13** on CPU.

```bash
git clone https://github.com/vs2806m/COMP1818-NIDS-Project.git
cd COMP1818-NIDS-Project
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt --extra-index-url https://download.pytorch.org/whl/cpu
```

### 2. Get the data

Download `UNSW_NB15_training-set.csv` and `UNSW_NB15_testing-set.csv` from
[UNSW-NB15 on Kaggle](https://www.kaggle.com/datasets/mrwellsdavid/unsw-nb15)
and place both files in [`data/`](data/). See
[`data/README.md`](data/README.md) for the expected file hashes.

### 3. Run

The script reads the dataset folder from the `DATA_DIR` environment variable
and writes all outputs to the current working directory, so run it from
inside `artifacts/`:

```bash
cd artifacts
DATA_DIR=../data python ../nids_pipeline.py | tee run_log.txt
```

Windows (PowerShell):

```powershell
cd artifacts
$env:DATA_DIR = "..\data"
python ..\nids_pipeline.py | Tee-Object run_log.txt
```

**Google Colab:** click the *Open in Colab* badge above, upload the two CSV
files to `/content`, and choose **Runtime → Run all**.

The multi-seed section retrains every model five times and accounts for most
of the runtime. You do not need to run the pipeline to inspect the results:
every output of the reported run is already saved in
[`artifacts/`](artifacts/).

---

## Research question

**Do three percentile-based rules improve an autoencoder-based network
intrusion detector on UNSW-NB15 once threshold confounds are isolated, and
how robust is any observed difference to threshold selection, duplicate
records and random-seed variation?**

- $H_0$: the hybrid's test MCC equals the rules-zeroed ablation's MCC within seed-to-seed variation.
- $H_1$: the two differ.

The 95% confidence interval of the paired five-seed difference contains
zero, so the experiments fail to reject $H_0$.

---

## Key results

Held-out test set: 82,332 records, attack prevalence 55.1%. No test labels
were used for any model, threshold or parameter selection.

| Model | MCC | FPR | Notes |
|---|---:|---:|---|
| Isolation Forest | 0.2907 | 0.0702 | |
| One-Class SVM | 0.6125 | 0.0648 | |
| Symbolic rules only | 0.2039 | 0.0274 | Rule configuration selected with validation labels |
| Deep autoencoder | 0.6259 | 0.0915 | |
| AE label-tuned control | 0.6395 | 0.0672 | Threshold selected with validation labels |
| Random-subspace ensemble | 0.5568 | 0.0788 | Unstable: 0.4692 ± 0.1071 across seeds |
| Logical OR (AE ∨ rules) | 0.6077 | 0.1161 | |
| Logical AND (AE ∧ rules) | 0.2515 | 0.0028 | |
| **Rules-zeroed ablation** | **0.6387** | 0.0627 | Autoencoder at threshold 1.667 |
| **Proposed hybrid** | **0.6374** | 0.0645 | Fusion selected with validation labels |
| Random Forest (supervised) | 0.7506 | 0.2743 | Reference only; trained with labels |

MCC = Matthews correlation coefficient; FPR = false-positive rate. Accuracy,
macro-F1, ROC-AUC, average precision, precision, recall and confusion counts
for every model are in
[`artifacts/final_experimental_results.csv`](artifacts/final_experimental_results.csv).

**What the results show**

- The rules change only **105** of 82,332 test decisions (37 attacks,
  68 normal). Those added alerts have 35% precision against the zeroed
  model's 93.1%, so aggregate precision falls slightly (0.9314 → 0.9296).
- Across five seeds the hybrid is statistically indistinguishable from the
  rules-zeroed control, the plain autoencoder and the label-tuned control
  ([`artifacts/multi_seed_paired_deltas.csv`](artifacts/multi_seed_paired_deltas.csv)).
- The Random Forest reaches a higher MCC only at a 27.4% false-positive rate,
  and its supervised setup is not comparable to the unsupervised models.

---

## Methodology

### Preprocessing

- Uses the official UNSW-NB15 training (175,341 records) and testing
  (82,332 records) partitions.
- Excludes `id`, `attack_cat`, the TCP base sequence numbers (`stcpb`,
  `dtcpb`) and the TTL features (`sttl`, `ct_state_ttl`).
- Applies $\log(1+x)$ to five heavy-tailed volume features.
- Fits the scaler and one-hot encoder on **normal training rows only**.
- Adds three binary indicators for `proto`/`service`/`state` values unseen in
  normal training traffic, giving a final input of **63 features**.

Feature exclusions are design choices, not proof that the excluded features
always constitute leakage or testbed artefacts.

### Internal splits

The training file is split with `GroupShuffleSplit`, grouping rows by a hash
of their model-visible feature vector (`pandas.util.hash_pandas_object`), into
four subsets: `train`, `val_es` (early stopping), `val_rules` (rule selection)
and `val_fusion` (thresholds and fusion). No model-visible feature vector
appears in more than one subset.

> **Caveat:** these internal splits do not remove the overlaps between the
> official training and testing partitions, so the primary benchmark is not
> leakage-free in the strict sense.

### Models

| Model | Role |
|---|---|
| Isolation Forest | Classical unsupervised baseline |
| One-Class SVM | Unsupervised support-estimation baseline |
| Deep autoencoder | Reconstruction-based anomaly detector |
| Random-subspace ensemble | Ensemble inspired by Kitsune/KitNET |
| Symbolic rules | Three hand-designed rules with percentile thresholds |
| Hybrid detector | Fusion of the autoencoder score and the rule flag |
| Random Forest | Supervised reference comparator |

**Autoencoder:** `63 → 32 → 16 → 32 → 63`, ReLU hidden activations, linear
output, MSE loss, Adam, trained on normal traffic only. The anomaly score
$S_{\mathrm{ML}}$ is reconstruction MSE divided by the 95th-percentile MSE of
normal `val_fusion` traffic.

**Random-subspace ensemble:** eight small autoencoders on disjoint feature
blocks plus an output autoencoder. It uses a random feature partition rather
than KitNET's correlation-based feature mapper, so it is
**Kitsune-inspired, not a faithful reproduction**.

### Symbolic rules

Each threshold $\tau$ is set at a percentile of normal training traffic.

| Rule | Condition | Intended behaviour |
|---|---|---|
| $R_1$ | rate $> \tau_{\text{rate}}$ and dur $< 0.05$ | Flood-like |
| $R_2$ | $\log(1+\text{sload}) - \log(1+\text{dload}) > \tau_{\text{ratio}}$ | Asymmetric outbound |
| $R_3$ | ct_srv_src $> \tau_{\text{ct}}$ and dur $< 0.05$ | Scan-like |

The rules are combined with logical OR:

$$S_{\mathrm{Sym}} = R_1 \lor R_2 \lor R_3$$

The configuration is selected on `val_rules` to maximise attack recall under a
2% FPR cap. **No configuration met the cap**, so the pipeline falls back to
the lowest-FPR configuration (validation FPR 3.27%). The rules are
experiment-specific heuristics, not an expert knowledge base.

### Hybrid fusion

$$S_{\mathrm{Hybrid}} = \alpha S_{\mathrm{ML}} + (1-\alpha) S_{\mathrm{Sym}} + \gamma \left(S_{\mathrm{ML}} \cdot S_{\mathrm{Sym}}\right)$$

A grid search on `val_fusion` selects $\alpha = 0.6$, $\gamma = 0.1$ and
$t = 1.0$. Substituting, the decision $S_{\mathrm{Hybrid}} > t$ becomes:

- $S_{\mathrm{ML}} > t/\alpha \approx 1.667$ when no rule fires;
- $S_{\mathrm{ML}} > (t-(1-\alpha))/(\alpha+\gamma) \approx 0.857$ when a rule fires.

The rules-zeroed ablation is therefore exactly the autoencoder thresholded at
1.667, which makes it a matched control for the hybrid.

### Evaluation

- Held-out test-set metrics; test labels are never used for selection.
- Five random seeds (42, 7, 123, 2024, 99) with paired per-seed differences.
- Rules-zeroed ablation and label-tuned threshold control. The control uses
  **validation** labels only and is not a deployable unsupervised model.
- Logical OR and logical AND comparisons.
- Per-attack-category recall.
- Unseen-category indicator ablation.
- Deduplicated-test sensitivity analysis.

---

## Dataset and data quality

### Source

The dataset was downloaded from
[UNSW-NB15 on Kaggle](https://www.kaggle.com/datasets/mrwellsdavid/unsw-nb15).
It was created by Moustafa and Slay at UNSW Canberra (see
[References](#references)). The data files are **not** included in this
repository.

In the Kaggle copy the two filenames are swapped relative to the official
split (`UNSW_NB15_training-set.csv` has 82,332 rows). The pipeline detects
this from the row counts and prints a warning, so the files can be used as
downloaded. The reported run used:

| Role | File | Rows | SHA-256 |
|---|---|---:|---|
| Training | `UNSW_NB15_testing-set.csv` | 175,341 | `bec7dd5ec88dc2a0ccc7a07879d338395ed7421750f675fd0339e07dfe0648fa` |
| Testing | `UNSW_NB15_training-set.csv` | 82,332 | `734fe6642edf758f7c94d7d9149426b49d202fe8e7bf0bef47392489c3c0a559` |

### Duplicate-record audit

| Audit (model-visible features) | Count |
|---|---:|
| Within-training duplicates | 74,303 |
| Within-testing duplicates | 28,386 |
| Cross-split overlaps | 1,305 |
| Conflicting-label groups in training | 229 |

The official split is kept for the primary benchmark. Removing duplicates
**within the test set only** (82,332 → 53,946 rows; attack prevalence
55.1% → 36.6%) lowers unsupervised MCC by 0.14–0.32. Because this also
changes prevalence and attack-category mix, the drop cannot be attributed
solely to duplicate removal.

---

## Repository structure

```text
COMP1818-NIDS-Project/
├── README.md                 # this file
├── LICENSE                   # MIT licence for the code (not the dataset)
├── requirements.txt          # pinned package versions
├── .gitignore
├── nids_pipeline.py          # full pipeline as a command-line script
├── nids_pipeline.ipynb       # Colab notebook with the saved run output
├── references_AI.bib         # bibliography of the LaTeX report
├── data/
│   └── README.md             # where to place the dataset CSVs
└── artifacts/
    ├── README.md             # description of every output file
    ├── run_log.txt           # console output of the reported run
    ├── *.csv, *.json         # results, configuration, thresholds
    ├── *.npy                 # split, subsample and ensemble indices
    ├── autoencoder_state_dict.pt
    ├── ensemble_state_dict.pt
    ├── preprocessor.joblib
    └── rf_preprocessor.joblib
```

---

## Reproducibility

- Seeds are fixed (`42` primary; `42, 7, 123, 2024, 99` for sensitivity) and
  PyTorch deterministic algorithms are enabled.
- Split indices, thresholds, selected rule and fusion parameters, and loss
  histories are saved in [`artifacts/`](artifacts/).
- Each run ends with a reload smoke test that reloads the saved preprocessor,
  autoencoder and ensemble and checks that they reproduce the test scores.
- A fresh rerun on Colab reproduced every saved output and every reported
  metric exactly.
- The saved preprocessors require **scikit-learn 1.6.1**, which is pinned in
  `requirements.txt`. Instructions for reloading the trained models are in
  [`artifacts/README.md`](artifacts/README.md).
- Small numerical differences are possible on other hardware or library
  versions; compare against [`artifacts/run_log.txt`](artifacts/run_log.txt).

---

## Limitations

- **Cross-split overlap:** 1,305 model-visible vectors appear in both official
  files and remain in the primary benchmark.
- **Single dataset, flow-level features:** results are confined to UNSW-NB15;
  Kitsune uses packet-derived features on different networks.
- **Offline training:** models are batch-trained, whereas Kitsune is online.
- **Small calibration set:** `val_fusion` contains 2,097 normal rows.
- **Autoencoder convergence:** some runs reach the 300-epoch cap.
- **Ensemble instability:** the primary-seed ensemble is an outlier run.
- **Distribution shift:** the Random Forest's 27% test FPR suggests a shift
  between the official partitions.
- **Rule and fusion scope:** no rule configuration met the 2% FPR cap, and
  only one fusion function form was tested.

---

## References

1. Mirsky, Y., Doitshman, T., Elovici, Y. and Shabtai, A. (2018). *Kitsune: An Ensemble of Autoencoders for Online Network Intrusion Detection.* NDSS. doi:10.14722/ndss.2018.23204
2. Moustafa, N. and Slay, J. (2016). *The evaluation of Network Anomaly Detection Systems: Statistical analysis of the UNSW-NB15 data set and the comparison with the KDD99 data set.* Information Security Journal: A Global Perspective, 25(1–3), 18–31.
3. Liu, F. T., Ting, K. M. and Zhou, Z.-H. (2008). *Isolation Forest.* IEEE ICDM, 413–422.
4. Schölkopf, B. et al. (2001). *Estimating the Support of a High-Dimensional Distribution.* Neural Computation, 13(7), 1443–1471.
5. Sommer, R. and Paxson, V. (2010). *Outside the Closed World: On Using Machine Learning for Network Intrusion Detection.* IEEE S&P, 305–316.
6. Arp, D. et al. (2022). *Dos and Don'ts of Machine Learning in Computer Security.* 31st USENIX Security Symposium, 3971–3988.
7. d'Avila Garcez, A. et al. (2019). *Neural-Symbolic Computing: An Effective Methodology for Principled Integration of Machine Learning and Reasoning.* Journal of Applied Logics, 6(4), 611–632.

---

## Licence and terms of use

The code is released under the MIT licence; see
[`LICENSE`](https://github.com/vs2806m/COMP1818-NIDS-Project/blob/main/LICENSE).
The UNSW-NB15 dataset is **not** covered by this licence and is not
redistributed here; use it under the terms set by its creators and cite
Moustafa and Slay when you do.

This repository was produced as individual coursework for COMP1818 at the
University of Greenwich.

## AI use declaration

Generative AI assisted in debugging pipeline code, conducting
methodological reviews, and polishing report drafts. To maintain strict
academic integrity, I independently verified every AI contribution by
re-running the complete experimental pipeline and cross-checking
all reported figures directly against generated output files. Through
this auditing, I identified and fixed significant AI errors, including
an unhandled id column that masked feature-vector duplicates and
incorrect threshold unit conversions within the multi-seed control
loop. Ultimately, this process taught me that while AI accelerates
prototyping, rigorous empirical validation and independent analytical
scrutiny remain irreplaceable in research.
[`artifacts/run_log.txt`](artifacts/run_log.txt).
