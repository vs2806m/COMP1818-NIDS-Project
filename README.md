<div align="center">

# Neuro-Symbolic Network Anomaly Detection on UNSW-NB15

### Three Percentile Rules Add No Measurable Benefit to an Autoencoder NIDS: A Controlled Test on UNSW-NB15

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
validation-tuned fusion of the autoencoder and the rules. Two controls, five
random seeds and a deduplicated-test analysis separate the effect of the rules
from threshold choice, duplicate records and training noise. The rules-zeroed
ablation is an exact matched control (the autoencoder at the fusion's own
effective threshold); the label-tuned threshold control is an approximate one.

> **Main finding**
>
> The hybrid detector reached a test MCC of **0.6374**, against **0.6387** for
> the rules-zeroed control. Across five seeds the paired difference was
> **+0.0004 ± 0.0028** (95% CI [−0.0031, +0.0039]). Under this protocol the
> three rules add **no measurable detection benefit**. The hybrid's +0.0115
> MCC gain over the plain autoencoder (seed 42) is a threshold adjustment and
> is not robust: the gain is positive in 4 of 5 seeds, but seed 123 reverses
> it by 0.071. A post-hoc TOST equivalence test with a ±0.01 MCC margin
> supports equivalence of the hybrid and the rules-zeroed control
> (p ≈ 0.0008), but not equivalence to the plain autoencoder (p ≈ 0.32) or the
> label-tuned control (p ≈ 0.43). See
> [Post-hoc calculations](#post-hoc-calculations) for how these are computed.

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
and writes all outputs to the current working directory. Run it from a **new,
empty folder** so that it does not overwrite the committed outputs in
`artifacts/`:

```bash
mkdir rerun && cd rerun
DATA_DIR=../data python ../nids_pipeline.py | tee run_log.txt
```

Windows (PowerShell):

```powershell
mkdir rerun; cd rerun
$env:DATA_DIR = "..\data"
python ..\nids_pipeline.py | Tee-Object run_log.txt
```

Compare the files in `rerun/` with those in [`artifacts/`](artifacts/).

**Google Colab:** click the *Open in Colab* badge above, upload the two CSV
files to `/content`, and choose **Runtime → Run all**. The notebook is the
version that produced the saved outputs; `nids_pipeline.py` differs from it
only by one unused assignment and a comment.

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

- $H_0$: the paired mean MCC difference between the hybrid and the
  rules-zeroed ablation is zero.
- $H_1$: the paired mean difference is not zero.
- Equivalence hypothesis: the true difference lies within ±0.01 MCC (margin
  chosen post hoc).

The 95% confidence interval of the paired five-seed difference
([−0.0031, +0.0039]) contains zero, so the experiments fail to reject $H_0$.
A TOST test supports equivalence within ±0.01 MCC against the rules-zeroed
ablation only (p ≈ 0.0008); equivalence is not established against the plain
autoencoder or the label-tuned control.

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
| Logical OR (AE ∨ rules) | 0.6077 | 0.1161 | Uses validation-selected rule configuration |
| Logical AND (AE ∧ rules) | 0.2515 | 0.0028 | Uses validation-selected rule configuration |
| **Rules-zeroed ablation** | **0.6387** | 0.0627 | AE at threshold 1.667, taken from the validation-tuned fusion |
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
- Across five seeds there is no significant difference between the hybrid
  and any of the rules-zeroed ablation, the plain autoencoder or the
  label-tuned control
  ([`artifacts/multi_seed_paired_deltas.csv`](artifacts/multi_seed_paired_deltas.csv)).
  Equivalence within ±0.01 MCC (TOST) is shown **only** against the
  rules-zeroed ablation (see [Post-hoc calculations](#post-hoc-calculations)).
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

Each threshold $\tau$ is set at a percentile of normal training traffic. The
selected configuration uses the 99th percentile and a duration bound of 0.05.

| Rule | Condition | Intended behaviour |
|---|---|---|
| $R_1$ | rate $> \tau_{\text{rate}}$ and dur $< 0.05$ | Flood-like |
| $R_2$ | $\log(1+\text{sload}) - \log(1+\text{dload}) > \tau_{\text{ratio}}$ | Asymmetric outbound |
| $R_3$ | ct_srv_src $> \tau_{\text{ct}}$ and dur $< 0.05$ | Scan-like |

The rules are combined with logical OR:

$$S_{\mathrm{Sym}} = R_1 \lor R_2 \lor R_3$$

The configuration is selected on `val_rules` to maximise attack recall under a
2% FPR cap. **No configuration met the cap**, so the pipeline falls back to
the lowest-FPR configuration (validation FPR 3.27%, recall 9.33%). The rules are
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
  Fusion parameters and rule thresholds are locked from seed 42 and the data
  split is fixed; only model initialisation varies across seeds.
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
55.1% → 36.6%) lowers unsupervised MCC by 0.14–0.31. Because this also
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
- The saved preprocessors require **scikit-learn 1.6.1**, which is pinned in
  `requirements.txt`. Instructions for reloading the trained models are in
  [`artifacts/README.md`](artifacts/README.md).
- Small numerical differences are possible on other hardware or library
  versions; compare against [`artifacts/run_log.txt`](artifacts/run_log.txt).
- `artifacts/rf_comparator.joblib` (about 37 MB) is not committed because of
  its size; the main run regenerates it.

### Where each report result comes from

| Report item | Source file in `artifacts/` |
|---|---|
| Table 2 (hyperparameters) | `experiment_config.json`, `baseline_config.json` |
| Table 3 (held-out test metrics) | `final_experimental_results.csv` |
| Table 3, five-seed MCC column (§5.2) | `multi_seed_sensitivity.csv` |
| Paired seed differences and 95% CIs | `multi_seed_paired_deltas.csv` |
| Selected rule percentile | `rule_validation_search.csv` |
| Selected fusion parameters (α, γ, t) | `fusion_validation_search.csv`, `thresholds.json` |
| Test-set deduplication analysis | `test_dedup_sensitivity.csv` |
| Unseen-category indicator ablation | `unseen_indicator_ablation.csv` |
| Per-attack-category results | `attack_category_results.csv` |
| Decisions changed by the rules (105) | `run_log.txt` |
| TOST equivalence results (§5.2) | `multi_seed_paired_deltas.csv` (see below) |
| Generic share of rule hits, ~86% (§5.3, §6.2) | `attack_category_results.csv` (see below) |
| Precision at 1% prevalence, ~9.8% (§6.4) | `final_experimental_results.csv` (see below) |

### Post-hoc calculations

Three figures in the report are calculated from the saved outputs rather than
printed by the pipeline. Each can be recomputed directly from the files above:

- **TOST equivalence test** (margin ±0.01 MCC, chosen post hoc). For each
  column *x* of `multi_seed_paired_deltas.csv`, the TOST p-value is the larger
  of `scipy.stats.ttest_1samp(x, -0.01, alternative="greater")` and
  `scipy.stats.ttest_1samp(x, 0.01, alternative="less")`. Results:
  hybrid − zeroed p ≈ 0.0008 (equivalent); hybrid − AE p ≈ 0.32;
  hybrid − control p ≈ 0.43 (neither equivalent).
- **Generic share of rule hits.** In `attack_category_results.csv`, rule hits
  per category = `Sample Count` × `rules_Recall`. Generic contributes
  18,871 × 0.3035 ≈ 5,727 of the 6,639 rule true positives (≈ 86%).
- **Precision at 1% attack prevalence.** Using `Recall` and `FPR` from
  `final_experimental_results.csv`: precision =
  (Recall × 0.01) / (Recall × 0.01 + FPR × 0.99). For the hybrid,
  (0.6952 × 0.01) / (0.6952 × 0.01 + 0.0645 × 0.99) ≈ 9.8%.

---

## Limitations

- **Cross-split overlap:** 1,305 model-visible vectors appear in both official
  files and remain in the primary benchmark.
- **Single dataset, flow-level features:** results are confined to UNSW-NB15;
  Kitsune uses packet-derived features on different networks.
- **Offline training:** models are batch-trained, whereas Kitsune is online.
- **Small calibration set:** `val_fusion` contains 2,097 normal rows.
- **Autoencoder convergence:** the autoencoder reaches the 300-epoch cap in
  2 of 5 seeds with the unseen-category indicators (seeds 42 and 2024) and in
  4 of 5 without them (seeds 42, 7, 123 and 99).
- **Ensemble instability:** ensemble MCC is 0.4692 ± 0.1071 across seeds.
  Seed 42's ensemble MCC (0.5568) is one of the best of the five, but its
  output autoencoder is an outlier (validation loss ≈ 0.0435, against below
  0.001 for every other seed). On the deduplicated test set its ROC-AUC is
  0.4825, below chance, while its average precision (also 0.4825) is above the
  attack-prevalence baseline of about 0.366.
- **Distribution shift:** the Random Forest's 27% test FPR suggests a shift
  between the official partitions.
- **Rule and fusion scope:** no rule configuration met the 2% FPR cap, and
  only one fusion function form was tested.
- **Rules cannot alert on their own:** in the searched grid the largest rule
  weight (1 − α = 0.6) is below the smallest threshold (t = 0.8), so rules
  can only lower the effective autoencoder threshold. This limits how much
  the symbolic component can change, and partly explains the null result.
- **Control threshold grid:** the label-tuned control searched 100
  percentiles of the validation normal scores plus three fixed values
  (1.5, 2.0, 3.0). It did not explicitly include the effective thresholds
  the fusion can reach (for example 1.667), so the match is approximate.
- **Rule percentile grid:** rule percentiles were searched only up to the
  99th, which is why the 2% FPR cap could not be met.
- **`dttl` retained:** `sttl` is dropped as a testbed-topology artefact, but
  `dttl`, which may carry the same artefact, is kept.
- **Multi-seed scope:** the multi-seed analysis varies only model
  initialisation; fusion and rule settings and the data split are fixed.

---

## References

These match [`references_AI.bib`](references_AI.bib), the bibliography of the report.

1. Bizzarri, A., Yu, C.-E., Jalaian, B., Riguzzi, F. and Bastian, N. D. (2025). *Neurosymbolic AI for network intrusion detection systems: A survey.* Journal of Information Security and Applications, 94, 104205.
2. Buczak, A. L. and Guven, E. (2016). *A survey of data mining and machine learning methods for cyber security intrusion detection.* IEEE Communications Surveys and Tutorials, 18(2), 1153–1176.
3. Chalapathy, R. and Chawla, S. (2019). *Deep learning for anomaly detection: A survey.* arXiv:1901.03407.
4. De Araujo-Filho, P. F., Naili, M., Kaddoum, G., Fapi, E. T. and Zhu, Z. (2023). *Unsupervised GAN-based intrusion detection system using temporal convolutional networks and self-attention.* IEEE Transactions on Network and Service Management, 20(4), 4951–4963.
5. Hinton, G. E. and Salakhutdinov, R. R. (2006). *Reducing the dimensionality of data with neural networks.* Science, 313(5786), 504–507.
6. Liu, F. T., Ting, K. M. and Zhou, Z.-H. (2008). *Isolation Forest.* IEEE International Conference on Data Mining, 413–422.
7. Mirsky, Y., Doitshman, T., Elovici, Y. and Shabtai, A. (2018). *Kitsune: An ensemble of autoencoders for online network intrusion detection.* NDSS.
8. Moustafa, N. and Slay, J. (2015). *UNSW-NB15: A comprehensive data set for network intrusion detection systems.* MilCIS, IEEE, 1–6.
9. Schölkopf, B., Platt, J. C., Shawe-Taylor, J., Smola, A. J. and Williamson, R. C. (2001). *Estimating the support of a high-dimensional distribution.* Neural Computation, 13(7), 1443–1471.
10. Sommer, R. and Paxson, V. (2010). *Outside the closed world: On using machine learning for network intrusion detection.* IEEE Symposium on Security and Privacy, 305–316.
11. Verkerken, M., D'Hooge, L., Sudyana, D., Lin, Y.-D., Wauters, T., Volckaert, B. and De Turck, F. (2023). *A novel multi-stage approach for hierarchical intrusion detection.* IEEE Transactions on Network and Service Management, 20(3), 3915–3929.
12. Zoghi, Z. and Serpen, G. (2024). *Building an intrusion detection system on UNSW-NB15: Reducing the margin of error to deal with data overlap and imbalance.* Concurrency and Computation: Practice and Experience, 36(25), e8242.

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

Generative AI tools (ChatGPT and Claude) were used to support code debugging,
methodological review, and report editing. They also supported my learning by
helping me examine model design, evaluation metrics, threshold calibration,
and the distinction between symbolic rules and learned anomaly scores. I
reviewed the suggested changes and checked reported values against the saved
experimental output files and run log. Reported numbers come from the saved
run in [`artifacts/run_log.txt`](artifacts/run_log.txt), or are calculated from
the saved outputs as described in
[Post-hoc calculations](#post-hoc-calculations). The final interpretation and
any remaining corrections are my responsibility.
