<div align="center">

# Neuro-Symbolic Network Anomaly Detection on UNSW-NB15

### Do Percentile-Based Rules Add Value to an Autoencoder NIDS? A Controlled Test with a Kitsune-Inspired Ensemble

[![Python](https://img.shields.io/badge/Python-3.13-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![PyTorch](https://img.shields.io/badge/Framework-PyTorch-EE4C2C?logo=pytorch&logoColor=white)](https://pytorch.org/)
[![Dataset](https://img.shields.io/badge/Dataset-UNSW--NB15-2563EB)](https://research.unsw.edu.au/projects/unsw-nb15-dataset)
[![Course](https://img.shields.io/badge/Course-COMP1818-6D28D9)](#)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

**University of Greenwich · COMP1818 Artificial Intelligence Applications**

*A controlled evaluation of whether simple percentile-based rules add measurable value to autoencoder-based network anomaly detection.*

</div>

---

## Overview

This repository contains the code, saved outputs and trained models for a
controlled evaluation of whether a small set of heuristic rules improves
unsupervised network anomaly detection on the **UNSW-NB15** dataset.

The study compares classical unsupervised detectors, a deep autoencoder,
a Kitsune-inspired random-subspace ensemble, three percentile-based rules,
and a validation-tuned hybrid score fusion. Matched ablations
(rules-zeroed and label-tuned threshold), five random seeds and a
deduplicated-test sensitivity analysis isolate the effect of the rules from
threshold choice, duplicate records and training noise.

> **Main finding**
>
> The hybrid detector reached a test MCC of **0.6374**, against **0.6387**
> for the rules-zeroed control. Across five seeds the paired difference was
> **+0.0004 ± 0.0028** (95% CI [−0.0031, +0.0039]). Under this protocol the
> three rules add **no measurable detection benefit**; the hybrid's apparent
> single-seed gain over the plain autoencoder is a threshold adjustment and
> does not replicate across seeds.

## At a glance

| | Reported result |
|---|---:|
| Official training partition | 175,341 records |
| Official testing partition | 82,332 records |
| Final model input | 63 features |
| Baseline autoencoder MCC | 0.6259 |
| Hybrid detector MCC | 0.6374 |
| Rules-zeroed control MCC | 0.6387 |
| Random seeds evaluated | 5 |

## Contents

- [Research question](#research-question)
- [Key findings](#key-findings)
- [Methodology](#methodology)
- [Experimental results](#experimental-results)
- [Dataset and data quality](#dataset-and-data-quality)
- [Repository structure](#repository-structure)
- [Installation](#installation)
- [Dataset setup](#dataset-setup)
- [Running the pipeline](#running-the-pipeline)
- [Reproducibility](#reproducibility)
- [Limitations](#limitations)
- [References](#references)
- [Terms of use](#terms-of-use)
- [AI use declaration](#ai-use-declaration)

---

## Research question

**Do three percentile-based rules improve the performance of an
autoencoder-based network intrusion detector on UNSW-NB15 once threshold
confounds are isolated, and how robust is any observed difference to
threshold selection, duplicate records and random-seed variation?**

- $H_0$: the hybrid's test MCC equals the rules-zeroed ablation's MCC within seed-to-seed variation.
- $H_1$: the two differ.

The 95% confidence interval of the paired five-seed difference contains
zero, so the experiments fail to reject $H_0$.

## Key findings

| Finding | Reported result |
|---|---|
| Baseline autoencoder | MCC **0.6259** · FPR **9.15%** |
| Hybrid detector | MCC **0.6374** · FPR **6.45%** |
| Rules-zeroed control | MCC **0.6387** · FPR **6.27%** |
| Five-seed paired difference | Hybrid − rules-zeroed: **+0.0004 ± 0.0028** |
| Decisions changed by the rules | **105** of 82,332 (37 attacks, 68 normal) |
| Symbolic rules alone | Recall **14.65%** · FPR **2.74%** |
| Random-subspace ensemble | Mean MCC **0.4692 ± 0.1071** (unstable) |
| Supervised Random Forest | MCC **0.7506** · FPR **27.43%** |

The hybrid outperforms the plain autoencoder in the primary run, but not
the rules-zeroed control or the label-tuned threshold control. The rules
change only 105 test decisions, and those added alerts have 35% precision
against the zeroed model's 93.1%, so aggregate precision falls slightly.

The Random Forest is a **supervised reference comparator**, not an
equivalent unsupervised baseline, because it is trained with labels.

---

## Methodology

### 1. Dataset and preprocessing

The project uses the official UNSW-NB15 training (175,341 records) and
testing (82,332 records) partitions. Preprocessing:

- Excludes `id`, `attack_cat`, the TCP base sequence numbers
  (`stcpb`, `dtcpb`) and the TTL features (`sttl`, `ct_state_ttl`).
- Applies $\log(1+x)$ to five heavy-tailed volume features.
- Fits the scaler and one-hot encoder on **normal training rows only**.
- Adds three binary indicators for `proto`/`service`/`state` values unseen
  in normal training traffic.
- Produces a final model input of **63 features**.

Feature exclusions are design choices. They should not be read as proof
that the excluded features always constitute leakage or testbed artefacts.

### 2. Internal data splits

The training file is split with `GroupShuffleSplit`, grouping rows by a
hash of their model-visible feature vector (`pandas.util.hash_pandas_object`).
The four internal subsets are `train`, `val_es` (early stopping),
`val_rules` (rule selection) and `val_fusion` (thresholds and fusion). No
model-visible feature vector appears in more than one subset.

> **Data-split caveat:** the internal splits do not remove the overlaps
> between the official training and testing partitions. The primary
> benchmark is therefore not leakage-free in the strict sense.

### 3. Detection models

| Model | Role |
|---|---|
| Isolation Forest | Classical unsupervised baseline |
| One-Class SVM | Unsupervised support-estimation baseline |
| Deep autoencoder | Reconstruction-based anomaly detection |
| Random-subspace ensemble | Ensemble inspired by Kitsune/KitNET |
| Symbolic rules | Three hand-designed rules with percentile thresholds |
| Hybrid detector | Fusion of the autoencoder score and rule flag |
| Random Forest | Supervised reference comparator |

**Autoencoder:** `63 → 32 → 16 → 32 → 63`, ReLU hidden activations, linear
output, MSE loss, Adam, trained on normal traffic only. The anomaly score
$S_{\mathrm{ML}}$ is reconstruction MSE divided by the 95th-percentile MSE
of normal `val_fusion` traffic.

**Random-subspace ensemble:** eight small autoencoders on disjoint feature
blocks plus an output autoencoder. Inspired by Kitsune/KitNET, but it uses a
random feature partition rather than KitNET's correlation-based feature
mapper, so it is **Kitsune-inspired, not a faithful reproduction**.

### 4. Symbolic rules

Three rules, each with a threshold $\tau$ set at a percentile of normal
training traffic:

| Rule | Condition | Intended behaviour |
|---|---|---|
| $R_1$ | rate $> \tau_{\text{rate}}$ and dur $< 0.05$ | Flood-like |
| $R_2$ | $\log(1+\text{sload}) - \log(1+\text{dload}) > \tau_{\text{ratio}}$ | Asymmetric outbound |
| $R_3$ | ct_srv_src $> \tau_{\text{ct}}$ and dur $< 0.05$ | Scan-like |

The rules are combined with logical OR:

$$S_{\mathrm{Sym}} = R_1 \lor R_2 \lor R_3$$

The configuration is selected on `val_rules` to maximise attack recall
under a 2% FPR cap. **No configuration met the cap**; the pipeline falls
back to the lowest-FPR configuration (validation FPR 3.27%). These rules are
experiment-specific heuristics, not an expert knowledge base.

### 5. Hybrid score fusion

$$S_{\mathrm{Hybrid}} = \alpha S_{\mathrm{ML}} + (1-\alpha) S_{\mathrm{Sym}} + \gamma \left(S_{\mathrm{ML}} \cdot S_{\mathrm{Sym}}\right)$$

A grid search on `val_fusion` selects $\alpha = 0.6$, $\gamma = 0.1$,
$t = 1.0$. Substituting, the decision $S_{\mathrm{Hybrid}} > t$ becomes:

- $S_{\mathrm{ML}} > t/\alpha \approx 1.667$ when no rule fires;
- $S_{\mathrm{ML}} > (t-(1-\alpha))/(\alpha+\gamma) \approx 0.857$ when a rule fires.

The rules-zeroed ablation is therefore exactly the autoencoder thresholded
at 1.667, which is what makes it a matched control.

### 6. Evaluation design

- Held-out test-set metrics (no test labels used for any selection).
- Five random seeds: 42, 7, 123, 2024, 99, with paired per-seed differences.
- Rules-zeroed ablation and label-tuned threshold control.
- Logical OR and logical AND comparisons.
- Per-attack-category recall.
- Unseen-category indicator ablation.
- Deduplicated-test sensitivity analysis.

The label-tuned control uses **validation** labels (never test labels) to
choose its threshold. It is a control, not a deployable unsupervised model.

---

## Experimental results

Test set: 82,332 records, attack prevalence 55.1%.

| Model | MCC | FPR |
|---|---:|---:|
| Isolation Forest | 0.2907 | 0.0702 |
| One-Class SVM | 0.6125 | 0.0648 |
| Symbolic rules only | 0.2039 | 0.0274 |
| Deep autoencoder | 0.6259 | 0.0915 |
| AE label-tuned control | 0.6395 | 0.0672 |
| Random-subspace ensemble | 0.5568 | 0.0788 |
| Logical OR | 0.6077 | 0.1161 |
| Logical AND | 0.2515 | 0.0028 |
| Rules-zeroed ablation | 0.6387 | 0.0627 |
| Hybrid detector | 0.6374 | 0.0645 |
| Supervised Random Forest | 0.7506 | 0.2743 |

MCC = Matthews correlation coefficient; FPR = false-positive rate. Full
metrics (accuracy, macro-F1, ROC-AUC, AP, precision, recall, confusion
counts) are in [`artifacts/final_experimental_results.csv`](artifacts/final_experimental_results.csv).

**Interpretation.** The hybrid improves on the plain autoencoder in this
single run, but not on the rules-zeroed or label-tuned controls, and the
gain does not replicate across seeds. The Random Forest achieves a higher
MCC only at a much higher FPR, and its supervised setup is not comparable
to the unsupervised models.

---

## Dataset and data quality

### Dataset source

- Official source: [UNSW-NB15 dataset page](https://research.unsw.edu.au/projects/unsw-nb15-dataset)
- Alternative mirror: [UNSW-NB15 on Kaggle](https://www.kaggle.com/datasets/mrwellsdavid/unsw-nb15)

Prefer the official source. The dataset is **not** included in this
repository; check its usage conditions and citation requirements before any
redistribution.

### Duplicate-record audit

| Audit (model-visible features) | Count |
|---|---:|
| Within-training duplicates | 74,303 |
| Within-testing duplicates | 28,386 |
| Cross-split overlaps | 1,305 |
| Conflicting-label groups in training | 229 |

The official split is kept for the primary benchmark. Deduplication of the
**test set only** is run as a sensitivity analysis (82,332 → 53,946 rows;
attack prevalence 55.1% → 36.6%). It lowers unsupervised MCC by 0.14–0.32,
but because it also changes prevalence and attack-category mix, the drop
cannot be attributed solely to duplicate removal.

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
├── data/
│   └── README.md             # where to place the dataset CSVs
├── artifacts/
│   ├── README.md             # description of every output file
│   ├── run_log.txt           # console output of the reported run
│   ├── *.csv, *.json         # results, configuration, thresholds
│   ├── *.npy                 # split, subsample and ensemble indices
│   ├── autoencoder_state_dict.pt
│   ├── ensemble_state_dict.pt
│   ├── preprocessor.joblib
│   └── rf_preprocessor.joblib
└── report/
    ├── README.md
    └── references_AI.bib     # bibliography of the LaTeX report
```

## Installation

Tested with **Python 3.13** on CPU.

```bash
git clone https://github.com/vs2806m/COMP1818-NIDS-Project.git
cd COMP1818-NIDS-Project
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt --extra-index-url https://download.pytorch.org/whl/cpu
```

## Dataset setup

1. Download `UNSW_NB15_training-set.csv` and `UNSW_NB15_testing-set.csv`
   from the official source.
2. Place both files in `data/`.

Some mirrors swap the two filenames. The pipeline detects this from the row
counts (175,341 training / 82,332 testing) and prints a warning, so the
files can be used as downloaded. It also prints the SHA-256 hash of each
file; the reported run used:

| File role | SHA-256 |
|---|---|
| Training (175,341 rows) | `bec7dd5ec88dc2a0ccc7a07879d338395ed7421750f675fd0339e07dfe0648fa` |
| Testing (82,332 rows) | `734fe6642edf758f7c94d7d9149426b49d202fe8e7bf0bef47392489c3c0a559` |

## Running the pipeline

The script reads the data folder from the `DATA_DIR` environment variable
(default `/content`, for Colab) and writes all outputs to the **current
working directory**. Run it from inside `artifacts/` so outputs land there:

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

**Google Colab:** open `nids_pipeline.ipynb`, upload the two CSVs to
`/content`, and run all cells.

The multi-seed section retrains every model five times and dominates the
runtime.

## Reproducibility

- Seeds are fixed (`42` primary; `42, 7, 123, 2024, 99` for sensitivity) and
  PyTorch deterministic algorithms are enabled.
- All split indices, thresholds, selected rule and fusion parameters and
  loss histories are saved in `artifacts/`.
- The run ends with a reload smoke test that reloads the saved preprocessor,
  autoencoder and ensemble and checks they reproduce the test scores.
- With the pinned versions in `requirements.txt` the reported numbers should
  reproduce; small numerical differences are possible across hardware or
  library versions.
- `artifacts/run_log.txt` is the console output of the run reported in the
  paper, for comparison.

## Limitations

- **Cross-split overlap:** 1,305 model-visible vectors appear in both
  official files and remain in the primary benchmark.
- **Single dataset, flow-level features:** results are confined to
  UNSW-NB15; Kitsune uses packet-derived features on different networks.
- **Offline training:** models are batch-trained, whereas Kitsune is online.
- **Small calibration set:** `val_fusion` has 2,097 normal rows.
- **Autoencoder convergence:** some runs hit the 300-epoch cap.
- **Ensemble instability:** the primary-seed ensemble is an outlier run.
- **Distribution shift:** the Random Forest's 27% test FPR suggests a shift
  between the official partitions.
- **Rule and fusion scope:** no rule configuration met the 2% FPR cap, and
  only one fusion function form was tested.

## References

1. Mirsky, Y., Doitshman, T., Elovici, Y. and Shabtai, A. (2018). *Kitsune: An Ensemble of Autoencoders for Online Network Intrusion Detection.* NDSS. doi:10.14722/ndss.2018.23204
2. Moustafa, N. and Slay, J. (2016). *The evaluation of Network Anomaly Detection Systems: Statistical analysis of the UNSW-NB15 data set and the comparison with the KDD99 data set.* Information Security Journal: A Global Perspective, 25(1–3), 18–31.
3. Liu, F. T., Ting, K. M. and Zhou, Z.-H. (2008). *Isolation Forest.* IEEE ICDM, 413–422.
4. Schölkopf, B. et al. (2001). *Estimating the Support of a High-Dimensional Distribution.* Neural Computation, 13(7), 1443–1471.
5. Sommer, R. and Paxson, V. (2010). *Outside the Closed World: On Using Machine Learning for Network Intrusion Detection.* IEEE S&P, 305–316.
6. Arp, D. et al. (2022). *Dos and Don'ts of Machine Learning in Computer Security.* 31st USENIX Security Symposium, 3971–3988.
7. d'Avila Garcez, A. et al. (2019). *Neural-Symbolic Computing: An Effective Methodology for Principled Integration of Machine Learning and Reasoning.* Journal of Applied Logics, 6(4), 611–632.

## Terms of use

The code is released under the MIT licence (see [`LICENSE`](LICENSE)). The
UNSW-NB15 dataset is **not** covered by this licence and is not
redistributed here; use it under the terms set by its creators and cite
Moustafa and Slay when you do.

This repository was produced as individual coursework for COMP1818 at the
University of Greenwich.

## AI use declaration

Generative AI was used to generate and debug parts of the pipeline code, to
review the experimental methodology, and to give feedback on drafts of the
report and this README. AI-assisted review flagged errors that I then
verified and corrected, including an `id` column that hid feature-vector
duplicates and incorrect threshold units in the multi-seed control. All
reported numbers come from the saved run in `artifacts/run_log.txt`.
