<div align="center">

# Neuro-Symbolic Network Anomaly Detection on UNSW-NB15

### Integrating Autoencoder-Based Detection with Knowledge-Based Rules

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![PyTorch](https://img.shields.io/badge/Framework-PyTorch-EE4C2C?logo=pytorch&logoColor=white)](https://pytorch.org/)
[![Dataset](https://img.shields.io/badge/Dataset-UNSW--NB15-2563EB)](https://research.unsw.edu.au/projects/unsw-nb15-dataset)
[![Course](https://img.shields.io/badge/Course-COMP1818-6D28D9)](https://www.gre.ac.uk/)

**University of Greenwich · COMP1818 Artificial Intelligence Applications**

*A controlled evaluation of whether simple symbolic rules add measurable value to autoencoder-based network anomaly detection.*

</div>

---

## Overview

This repository contains the implementation and experimental pipeline for evaluating whether a small set of heuristic symbolic rules improves unsupervised network anomaly detection on the **UNSW-NB15** dataset.

The study compares classical unsupervised detectors, a deep autoencoder, a Kitsune-inspired random-subspace ensemble, three percentile-based symbolic rules, and a hybrid score-fusion approach. Matched ablations and multi-seed experiments investigate the effects of rule activation, threshold calibration, duplicate records, and random-seed variation.

> **Main finding**
>
> In the reported single-seed experiment, the hybrid detector achieved an MCC of **0.6374**, compared with **0.6387** for the rules-zeroed control. Across five seeds, the reported paired difference was **+0.0004 ± 0.0028**. These results do **not** demonstrate a robust performance advantage from adding the symbolic rules.

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

---

## Research question

**Do manually specified percentile-based symbolic rules improve the performance of an autoencoder-based network intrusion detection system on UNSW-NB15, and how robust are the observed results to threshold selection, duplicate records, and random-seed variation?**

The evaluation focuses on detection quality, false-positive behaviour, and the consistency of observed performance differences across experimental conditions.

## Key findings

The following findings reflect the project's reported experiments.

| Finding | Reported result |
|---|---|
| Baseline autoencoder | MCC: **0.6259** · FPR: **9.15%** |
| Hybrid detector | MCC: **0.6374** · FPR: **6.45%** |
| Rules-zeroed control | MCC: **0.6387** · FPR: **6.27%** |
| Five-seed paired difference | Hybrid − rules-zeroed: **+0.0004 ± 0.0028** |
| Symbolic rules alone | Attack recall: **14.65%** · FPR: **2.74%** |
| Random-subspace ensemble | Mean MCC: **0.4692 ± 0.1071** |
| Supervised Random Forest | MCC: **0.7506** · FPR: **27.43%** |

The hybrid detector performs slightly better than the baseline autoencoder in the reported primary run, but it does not outperform the rules-zeroed control. The multi-seed difference is small and does not establish a robust advantage attributable to symbolic-rule integration.

The Random Forest is included as a **supervised reference comparator**, not as a directly equivalent unsupervised baseline, because it uses labelled training data.

---

## Methodology

### 1. Dataset and preprocessing

The project uses the official UNSW-NB15 training and testing partitions.

| Partition | Records |
|---|---:|
| Training | 175,341 |
| Testing | 82,332 |

Preprocessing includes:

- Excluding identifiers and selected features, including attack-category labels, TCP sequence numbers, and selected TTL-related features.
- Applying log transformations to selected heavy-tailed volume features.
- Fitting the scaler and one-hot encoder using normal training rows only.
- Creating three binary indicators for categorical values unseen in normal training traffic.
- Producing a final model input dimension of **63 features**.

Feature exclusions are experimental design choices; they should not be interpreted as proof that the excluded features always constitute leakage or testbed artefacts.

### 2. Internal data splits

The reported pipeline uses `GroupShuffleSplit` with groups derived from SHA-256 hashes of model-visible feature vectors. The internal subsets are:

- `train`
- `val_es`
- `val_rules`
- `val_fusion`

The intention is to prevent identical model-visible feature vectors from crossing these internal subsets.

> **Data-split caveat:** These internal splits do not remove the reported overlaps between the official training and testing partitions. The primary benchmark should therefore not be described as completely leakage-free.

### 3. Detection models

| Model | Role |
|---|---|
| Isolation Forest | Classical unsupervised baseline |
| One-Class SVM | Unsupervised distribution-support baseline |
| Deep autoencoder | Reconstruction-based anomaly detection |
| Random-subspace ensemble | Ensemble inspired by Kitsune/KitNET |
| Symbolic rules | Three manually specified percentile-based conditions |
| Hybrid detector | Fusion of the autoencoder score and rule indicator |
| Random Forest | Supervised reference comparator |

#### Autoencoder architecture

The reported layer dimensions are:

```text
63 → 32 → 16 → 32 → 63
```

The autoencoder uses ReLU hidden-layer activations, a linear output layer, mean squared error (MSE) loss, and the Adam optimiser. It is trained on normal traffic to reconstruct input features.

The anomaly score is calculated as reconstruction MSE normalised by the 95th-percentile reconstruction MSE of normal validation traffic.

#### Random-subspace ensemble

The ensemble uses eight small autoencoders and an output autoencoder. Its design is inspired by Kitsune/KitNET, an ensemble-based approach to online network intrusion detection.

This implementation uses **random feature partitioning**, rather than KitNET's correlation-based feature-mapping procedure. It should therefore be regarded as *Kitsune-inspired*, not as a faithful reproduction of the original system.

### 4. Symbolic rules

Three heuristic percentile-based rules flag potentially suspicious traffic:

1. Flood-like behaviour.
2. Asymmetric outbound traffic.
3. Scan-like behaviour.

The rules are combined using logical OR:

\[
S_{\mathrm{Sym}} = R_1 \lor R_2 \lor R_3
\]

The rules are selected using a validation procedure with a target false-positive-rate cap of **2%**. The reported fallback configuration does not achieve that cap, so the target should not be interpreted as a guaranteed operating point.

These conditions are experiment-specific heuristics, not a comprehensive expert knowledge base.

### 5. Hybrid score fusion

The hybrid detector combines the machine-learning score with the symbolic-rule indicator:

\[
S_{\mathrm{Hybrid}} =
\alpha S_{\mathrm{ML}} +
(1-\alpha)S_{\mathrm{Sym}} +
\gamma(S_{\mathrm{ML}} \cdot S_{\mathrm{Sym}})
\]

where:

- \(S_{\mathrm{ML}}\) is the autoencoder anomaly score.
- \(S_{\mathrm{Sym}}\) is the binary symbolic-rule indicator.
- \(\alpha\) weights the machine-learning score.
- \(\gamma\) controls the interaction term.

The reported configuration uses:

\[
\alpha = 0.6,\qquad \gamma = 0.1,\qquad t = 1.0
\]

When no symbolic rule fires, \(S_{\mathrm{Sym}}=0\), and the fusion score becomes:

\[
S_{\mathrm{Hybrid}} = 0.6S_{\mathrm{ML}}
\]

Under the reported decision threshold \(t=1.0\), this corresponds to:

\[
S_{\mathrm{ML}} > \frac{1}{0.6} \approx 1.667
\]

This algebra shows that the hybrid decision rule changes the effective threshold when no rule fires. It does not, by itself, prove that all observed performance differences are caused by threshold calibration.

### 6. Evaluation design

The reported evaluation includes:

- Held-out test-set metrics.
- Five random seeds: `42`, `7`, `123`, `2024`, and `99`.
- Attack-category recall.
- Duplicate-record sensitivity analysis.
- Rules-zeroed ablation.
- Label-tuned threshold control.
- Logical OR and logical AND comparisons.

The label-tuned control uses test labels for a diagnostic comparison. It should not be interpreted as a deployable unsupervised model or a strictly label-independent benchmark.

---

## Experimental results

The table below reproduces the reported primary test-set results.

**Test set:** 82,332 records, with reported attack prevalence of 55.1%.

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
| **Hybrid detector** | **0.6374** | **0.0645** |
| Supervised Random Forest | 0.7506 | 0.2743 |

*Abbreviations: MCC = Matthews correlation coefficient; FPR = false-positive rate.*

### Interpretation

The hybrid detector improves on the baseline autoencoder in this single run, but it does not improve on the rules-zeroed control or the label-tuned threshold control.

Across five seeds, the reported paired difference between the hybrid and rules-zeroed control is small. The results do not establish a robust advantage attributable to symbolic-rule integration.

The Random Forest achieves a higher MCC but has a substantially higher FPR. Its supervised training setup also differs from the unsupervised approaches.

---

## Dataset and data quality

### Dataset source

The project uses the official UNSW-NB15 training and testing CSV files.

- **Official source:** [UNSW-NB15 dataset page](https://research.unsw.edu.au/projects/unsw-nb15-dataset)
- **Alternative mirror:** [UNSW-NB15 on Kaggle](https://www.kaggle.com/datasets/mrwellsdavid/unsw-nb15)

The official UNSW page identifies the training partition as 175,341 records and the testing partition as 82,332 records. It also requests citation of the dataset's associated publications for academic use. [Source](https://research.unsw.edu.au/projects/unsw-nb15-dataset)

Prefer the official source where possible. Check the dataset's usage conditions and citation requirements before redistribution or commercial use.

### Duplicate-record audit

The reported model-visible duplicate audit identifies:

| Audit | Reported count |
|---|---:|
| Within-training duplicates | 74,303 |
| Within-testing duplicates | 28,386 |
| Cross-split overlaps | 1,305 |

The project retains the official split for its primary benchmark and conducts deduplication as a sensitivity analysis.

Deduplication changes the number of evaluated records and can change class prevalence and attack-category composition. Performance reductions after deduplication therefore cannot automatically be attributed solely to duplicate removal.

The reported overlap between the official training and testing partitions means the primary benchmark should not be described as entirely free from cross-split overlap. The group-aware internal splits address a separate issue: repeated model-visible feature vectors crossing the internal validation subsets.

---

## Repository structure

The intended project structure is shown below. Update it to match the files actually committed to GitHub.

```text
COMP1818-NIDS-Project/
├── nids_pipeline.py
├── requirements.txt
├── README.md
├── references_AI.bib
└── artifacts/
    ├── final_experimental_results.csv
    ├── multi_seed_sensitivity.csv
    ├── experiment_config.json
    ├── autoencoder_state_dict.pt
    └── ...
```

| File or directory | Purpose |
|---|---|
| `nids_pipeline.py` | Main experimental pipeline |
| `requirements.txt` | Python dependencies |
| `references_AI.bib` | BibTeX bibliography |
| `artifacts/` | Generated results, configurations, and model weights, if produced |

Do not commit large generated files or dataset files unless necessary. If artefacts are not committed, document how to regenerate them.

---

## Installation

### Requirements

- Python 3.10+ (or another version supported by the dependencies)
- PyTorch
- scikit-learn
- pandas
- NumPy
- joblib

Create a virtual environment and install the dependencies.

**Linux / macOS**

```bash
git clone https://github.com/vs2806m/COMP1818-NIDS-Project.git
cd COMP1818-NIDS-Project

python3 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
pip install -r requirements.txt
```

**Windows PowerShell**

```powershell
git clone https://github.com/vs2806m/COMP1818-NIDS-Project.git
cd COMP1818-NIDS-Project

py -m venv .venv
.\.venv\Scripts\Activate.ps1

python -m pip install --upgrade pip
pip install -r requirements.txt
```

The dependency list and Python version should match the environment in which the pipeline has actually been tested.

## Dataset setup

1. Download `UNSW_NB15_training-set.csv` and `UNSW_NB15_testing-set.csv` from the [official dataset page](https://research.unsw.edu.au/projects/unsw-nb15-dataset) or the [Kaggle mirror](https://www.kaggle.com/datasets/mrwellsdavid/unsw-nb15).
2. Place the CSV files in the location expected by the script.
3. If supported by the implementation, set `DATA_DIR` to the directory containing the files.

**Linux / macOS**

```bash
export DATA_DIR="/path/to/csv/directory"
```

**Windows PowerShell**

```powershell
$env:DATA_DIR = "C:\path\to\csv\directory"
```

Verify that `nids_pipeline.py` reads this environment variable before relying on it. If the script uses hard-coded paths or a configuration file instead, update the paths accordingly.

> **Data note:** Do not commit raw dataset CSV files to the public repository unless you have confirmed that redistribution is permitted and appropriate.

## Running the pipeline

From the repository root:

```bash
python nids_pipeline.py
```

This command assumes the script supports execution without additional arguments and that the dataset paths have been configured correctly.

The described pipeline includes data auditing, internal splitting, model training, rule selection, fusion calibration, ablation experiments, multi-seed evaluation, and artefact generation. Check the script output to confirm the exact execution stages and output locations.

---

## Reproducibility

The project reports the following reproducibility measures:

- **Primary random seed:** `42`
- **Sensitivity seeds:** `42`, `7`, `123`, `2024`, `99`
- Group-aware internal splits based on SHA-256 hashes of model-visible feature vectors.
- Preprocessing fitted on normal training rows only.
- Recorded fusion and rule-threshold settings.
- Saved model weights and experiment artefacts, where generated.

For a reproducible release, also record the Python version, dependency versions, dataset source, and code revision. Preserve per-seed results and evaluation configuration so the reported comparisons can be independently checked.

## Limitations

1. **Heuristic rules:** The percentile-based rules are manually specified conditions, not a comprehensive expert knowledge base.
2. **Threshold sensitivity:** Decision thresholds influence MCC, recall, and FPR. Performance shifts do not necessarily indicate unique semantic reasoning from the rules.
3. **Cross-split overlap:** Model-visible feature overlaps are reported in the official train/test partitions.
4. **Duplicate sensitivity:** Deduplication changes the evaluation distribution and potentially the class and attack-category balance.
5. **Uneven attack-category recall:** The reported Shellcode recall is 0%, showing that aggregate metrics can conceal category-specific weaknesses.
6. **Limited seed count:** Five seeds provide a useful sensitivity analysis but do not constitute exhaustive statistical verification.
7. **Dataset generalisation:** Results on UNSW-NB15 do not guarantee equivalent performance on contemporary live network traffic.
8. **Architecture differences:** The random-subspace ensemble is Kitsune-inspired but does not reproduce KitNET's original correlation-based feature mapper.
9. **Operational suitability:** Benchmark FPR alone does not establish operational viability; deployment also requires consideration of alert volumes, network throughput, and operational constraints.

---

## References

1. Moustafa, N., & Slay, J. (2015). *UNSW-NB15: A comprehensive data set for network intrusion detection systems.* Military Communications and Information Systems Conference (MilCIS). [https://doi.org/10.1109/MILCIS.2015.7348942](https://doi.org/10.1109/MILCIS.2015.7348942)
2. Moustafa, N., & Slay, J. (2016). *The evaluation of Network Anomaly Detection Systems: Statistical analysis of the UNSW-NB15 data set and the comparison with the KDD99 data set.* Information Security Journal: A Global Perspective, 25(1–3), 18–31. [https://doi.org/10.1080/19393555.2015.1125974](https://doi.org/10.1080/19393555.2015.1125974)
3. Mirsky, Y., Doitshman, T., Elovici, Y., & Shabtai, A. (2018). *Kitsune: An Ensemble of Autoencoders for Online Network Intrusion Detection.* Network and Distributed System Security Symposium (NDSS). [https://doi.org/10.14722/ndss.2018.23204](https://doi.org/10.14722/ndss.2018.23204)
4. Liu, F. T., Ting, K. M., & Zhou, Z.-H. (2008). *Isolation Forest.* IEEE International Conference on Data Mining (ICDM), 413–422. [https://doi.org/10.1109/ICDM.2008.17](https://doi.org/10.1109/ICDM.2008.17)
5. Schölkopf, B., Platt, J. C., Shawe-Taylor, J., Smola, A. J., & Williamson, R. C. (2001). *Estimating the Support of a High-Dimensional Distribution.* Neural Computation, 13(7), 1443–1471. [https://doi.org/10.1162/089976601750264965](https://doi.org/10.1162/089976601750264965)
6. Sommer, R., & Paxson, V. (2010). *Outside the Closed World: On Using Machine Learning for Network Intrusion Detection.* IEEE Symposium on Security and Privacy, 305–316. [https://doi.org/10.1109/SP.2010.25](https://doi.org/10.1109/SP.2010.25)
7. d'Avila Garcez, A., Gori, M., Lamb, L. C., Serafini, L., Spranger, M., & Tran, S. N. (2019). *Neural-symbolic computing: An effective methodology for principled integration of machine learning and reasoning.* Journal of Applied Logics, 6(4), 611–632.

Complete BibTeX entries are maintained in `references_AI.bib`. Verify that every reference listed here is present in the bibliography file and that the metadata matches the cited publication.

---

## Terms of use

This repository contains academic coursework material created for COMP1818 at the University of Greenwich.

**Dataset terms:** The UNSW-NB15 dataset is provided by the Australian Centre for Cyber Security for academic research use, subject to the conditions stated by the dataset maintainers. The official dataset page asks users to cite the associated publications and states that commercial use requires agreement from the authors. Review the [official UNSW-NB15 dataset page](https://research.unsw.edu.au/projects/unsw-nb15-dataset) for the current terms.

**Code licence:** No code licence is declared in this README. If you want others to reuse the code, choose and add an appropriate `LICENSE` file before adding a licence badge or describing the repository as open source.
