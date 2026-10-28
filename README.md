# COMP1818-NIDS-Project

# Neuro-Symbolic Network Anomaly Detection on UNSW-NB15

A reproducible experimental framework for evaluating whether combining deep autoencoder-based anomaly detection with explicit symbolic rules improves network intrusion detection on the **UNSW-NB15 benchmark**.

This repository accompanies the project:

**Symbolic Rules Add No Measurable Benefit: A Controlled Evaluation of Neuro-Symbolic Network Intrusion Detection**

The study compares unsupervised anomaly detection baselines, a deep autoencoder, a Kitsune-inspired random-subspace ensemble, percentile-based symbolic rules, and a hybrid fusion model. Its central finding is that the three evaluated rules did not demonstrate a measurable improvement over a matched autoencoder control under the experimental protocol used.

> **Research scope:** The findings apply to the specific rules, models, data preparation and evaluation protocol in this repository. They should not be interpreted as evidence that symbolic reasoning or neuro-symbolic systems are generally ineffective.

---

## Table of Contents

- [Research Question](#research-question)
- [Key Findings](#key-findings)
- [Pipeline Overview](#pipeline-overview)
- [Experimental Results](#experimental-results)
- [Methodology](#methodology)
- [Dataset and Data Quality](#dataset-and-data-quality)
- [Repository Structure](#repository-structure)
- [Installation](#installation)
- [Dataset Setup](#dataset-setup)
- [Running the Pipeline](#running-the-pipeline)
- [Reproducibility and Generated Artifacts](#reproducibility-and-generated-artifacts)
- [Limitations](#limitations)
- [Citation](#citation)
- [License](#license)

## Research Question

**Do explicit symbolic rules improve an autoencoder-based network anomaly detector when the decision threshold is controlled?**

The experiments investigate this question through comparisons between:

1. Classical unsupervised baselines.
2. A deep autoencoder trained on normal traffic.
3. A random-subspace autoencoder ensemble inspired by KitNET.
4. Three heuristic rules derived from normal-feature percentiles.
5. A hybrid model combining reconstruction-based anomaly scores with symbolic rule outputs.
6. Matched ablations, multiple random seeds, and a supervised reference model.

The aim is not simply to maximise a benchmark metric. It is to determine whether the symbolic layer contributes useful detection performance beyond what can be achieved through the autoencoder and its decision threshold.

## Key Findings

### 1. The evaluated rules did not demonstrate additional detection value

The primary held-out test results report:

| Model | Matthews correlation coefficient (MCC) |
|---|---:|
| Deep Autoencoder | 0.6259 |
| Hybrid with rules zeroed | 0.6387 |
| Proposed Neuro-Symbolic Hybrid | 0.6374 |

The hybrid does not outperform its rules-zeroed control in this evaluation. Across five seeds, the reported mean paired MCC difference is **+0.0004**, with a paired-difference standard deviation of **0.0028**.

This is evidence that the tested rules provided little observed benefit under this setup. The five-seed experiment is small, so the result should not be interpreted as a formal equivalence test or as a general conclusion about all symbolic methods.

### 2. Fusion changes the effective decision threshold

The fusion model combines a continuous machine-learning score with a binary symbolic indicator:

\[
S_{\mathrm{Hybrid}}
=\alpha S_{\mathrm{ML}}
+(1-\alpha)S_{\mathrm{Sym}}
+\gamma(S_{\mathrm{ML}}S_{\mathrm{Sym}})
\]

where \(S_{\mathrm{Sym}}\in\{0,1\}\).

For the reported configuration, \(\alpha=0.6\), \(\gamma=0.1\), and \(t=1.0\). When no rule fires, the decision condition reduces to \(S_{\mathrm{ML}}>1.667\). When a rule fires, the threshold becomes \(S_{\mathrm{ML}}>0.857\).

This shows that the rules can change which samples cross the decision threshold. The matched rules-zeroed control helps investigate whether the observed performance comes from the rule layer or from the changed operating point.

### 3. Duplicate records complicate benchmark interpretation

The data audit reports **1,305 model-visible feature-vector overlaps** between the official training and test files.

Removing model-visible duplicate rows from the test evaluation reduces reported MCC for the unsupervised models by approximately 0.14–0.32. However, the deduplicated test set also has a different attack prevalence and attack-category composition. The change therefore cannot be attributed to duplicates alone.

### 4. The ensemble is unstable across random seeds

The random-subspace ensemble reports a primary-run MCC of 0.5568, while its five-seed mean is 0.4692 with a standard deviation of 0.1071. This variability warrants caution when interpreting the ensemble's performance.

### 5. Supervised performance has a different operating trade-off

The supervised Random Forest comparator reports MCC 0.7506, but its false-positive rate is 27.43%. Because it uses labelled training data and operates at a different error trade-off, it is a reference point rather than a directly equivalent unsupervised alternative.

## Pipeline Overview

The experimental workflow is organised as follows:

```text
UNSW-NB15 training and test CSV files
                 |
                 v
       Data integrity checks
       and duplicate audit
                 |
                 v
       Feature preprocessing
       and transformation
                 |
                 v
    Group-aware internal splits
                 |
        +--------+---------+
        |                  |
        v                  v
  Model training      Rule selection
  and early stopping  on validation data
        |                  |
        +--------+---------+
                 |
                 v
       Fusion calibration
       and matched ablations
                 |
                 v
       Held-out test evaluation
                 |
                 v
     Multi-seed and deduplication
          sensitivity analysis
                 |
                 v
      Results and audit artifacts
```

The internal group-aware splits are intended to prevent identical model-visible feature vectors from appearing across internal subsets. They do **not** remove all overlap between the original official training and test files.

## Experimental Results

The following table summarises the reported primary evaluation on the official UNSW-NB15 test split of 82,332 records.

| Model | Accuracy | Macro-F1 | MCC | ROC AUC | Precision | Recall | FPR |
|---|---:|---:|---:|---:|---:|---:|---:|
| Isolation Forest | 0.5849 | 0.5570 | 0.2907 | 0.7863 | 0.8411 | 0.3033 | 0.0702 |
| One-Class SVM | 0.7876 | 0.7870 | 0.6125 | 0.8246 | 0.9265 | 0.6671 | 0.0648 |
| Symbolic Rules Only | 0.5177 | 0.4475 | 0.2039 | — | 0.8674 | 0.1465 | 0.0274 |
| Deep Autoencoder | 0.8021 | 0.8021 | 0.6259 | 0.8731 | 0.9055 | 0.7152 | 0.0915 |
| Random-Subspace Ensemble | 0.7561 | 0.7549 | 0.5568 | 0.7450 | 0.9062 | 0.6214 | 0.0788 |
| Hybrid with Rules Zeroed | 0.8035 | 0.8032 | 0.6387 | 0.8731 | 0.9314 | 0.6943 | 0.0627 |
| Proposed Neuro-Symbolic Hybrid | 0.8032 | 0.8029 | 0.6374 | 0.8720 | 0.9296 | 0.6952 | 0.0645 |
| *Random Forest (supervised)* | *0.8690* | *0.8625* | *0.7506* | *0.9785* | *0.8149* | *0.9859* | *0.2743* |

**Metric notes**

- **MCC:** Matthews correlation coefficient, a measure of binary classification quality that accounts for all four confusion-matrix outcomes.
- **Macro-F1:** The unweighted mean of the F1 scores for the two classes.
- **ROC AUC:** Area under the receiver operating characteristic curve, calculated from continuous model scores where available.
- **FPR:** False-positive rate, calculated as \(FP/(FP+TN)\).
- **AP:** Average precision; it is reported in the full experimental results but omitted from this summary table for brevity.
- **—:** Not applicable to the binary-only outputs in this comparison.

These figures are the reported primary-run results, not independently reproduced results. See the generated result files and experimental configuration for the detailed evaluation records.

## Methodology

### Dataset and preprocessing

The project uses the official UNSW-NB15 training and testing partitions:

- Training set: 175,341 records.
- Test set: 82,332 records.
- Binary labels: normal (0) and attack (1).

The reported preprocessing includes:

- Excluding identifiers and the target/category labels from model inputs.
- Excluding selected TCP sequence-number and TTL-related features.
- Applying `log1p` transformations to selected heavy-tailed volume features.
- Fitting `MinMaxScaler` and `OneHotEncoder` on normal training data.
- Auditing duplicate feature vectors and conflicting-label groups.

The decision to exclude TTL-related features is motivated by the observed distribution in this dataset. It is a modelling choice, not proof that these features are universally invalid for intrusion detection.

### Deep autoencoder

The main autoencoder is an MLP with architecture:

```text
63 -> 32 -> 16 -> 32 -> 63
```

It is trained on normal traffic using reconstruction loss. Anomaly scores are based on reconstruction error, with a decision threshold derived from normal validation data.

### Random-subspace ensemble

The ensemble uses eight autoencoders on disjoint random feature subsets, followed by an output autoencoder. It is **inspired by KitNET**, the anomaly-detection component of Kitsune, but does not reproduce the original packet-level feature extractor or correlation-based feature mapper.

### Symbolic rules

Three heuristic rules are derived from normal-feature percentiles. Rule configurations are evaluated on a dedicated validation subset under an intended false-positive-rate constraint. The reported search did not find a configuration satisfying the 2% FPR cap, so the pipeline selected the lowest-FPR available configuration.

These are percentile-based heuristics rather than a complete expert knowledge base or a signature-based intrusion detection system.

### Hybrid fusion and ablations

The hybrid combines the machine-learning anomaly score with a binary indicator representing whether any symbolic rule fires. The evaluation includes a rules-zeroed control, a label-tuned threshold control, logical rule combinations, and multiple random seeds.

The supervised Random Forest is included as a labelled-data reference, not as an unsupervised baseline.

## Dataset and Data Quality

UNSW-NB15 contains duplicate records and shared feature vectors across the official partitions. This repository reports both raw-schema and model-visible duplicate audits.

The primary benchmark retains the official test partition for comparability with the reported experiment. A separate deduplication sensitivity analysis removes repeated model-visible test vectors.

**Important interpretation caveat:** deduplication changes the test-set size, attack prevalence and attack-category distribution. Results on the deduplicated subset are therefore a sensitivity analysis, not a clean estimate of performance inflation caused by duplicates alone.

The benchmark's attack prevalence is approximately 55.1%, which should not be assumed to reflect the class distribution of a deployed network. Operational alert volume and false-positive burden can differ substantially under real-world traffic distributions.

## Repository Structure

The principal files and expected outputs are organised as follows:

```text
.
├── nids_pipeline.py
├── requirements.txt
├── README.md
├── LICENSE
│
├── Saved models and preprocessing
│   ├── autoencoder_state_dict.pt
│   ├── ensemble_state_dict.pt
│   ├── preprocessor.joblib
│   ├── rf_comparator.joblib
│   ├── feature_names.json
│   └── thresholds.json
│
├── Experimental results
│   ├── final_experimental_results.csv
│   ├── attack_category_results.csv
│   ├── multi_seed_sensitivity.csv
│   ├── multi_seed_paired_deltas.csv
│   ├── unseen_indicator_ablation.csv
│   ├── test_dedup_sensitivity.csv
│   ├── rule_validation_search.csv
│   └── fusion_validation_search.csv
│
└── Data manifests and split indices
    ├── data_manifest.json
    ├── experiment_config.json
    ├── baseline_config.json
    ├── train_indices.npy
    ├── val_es_indices.npy
    ├── val_rules_indices.npy
    ├── val_fusion_indices.npy
    └── ensemble_subspace_indices.npy
```

The filenames above describe the reported project artifacts. Confirm the current pipeline's actual output paths and generated files before relying on this list; model weights, indices and result files should be regenerated from the same final code version if they are used to support the reported results.

## Installation

### Requirements

- Python 3.10 or newer.
- Packages listed in `requirements.txt`.
- CPU or compatible CUDA-enabled PyTorch environment.
- Sufficient memory and storage for the dataset, model training and generated artifacts.

A GPU is optional if the code and installed dependencies support CPU execution.

### 1. Clone the repository

```bash
git clone https://github.com/vs2806m/COMP1818-NIDS-Project.git
cd COMP1818-NIDS-Project
```

### 2. Create and activate a virtual environment

**Linux or macOS**

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

**Windows PowerShell**

```powershell
py -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

For repeatable results, use a documented Python version and record the installed package versions. The actual dependencies and supported versions should be taken from the repository's `requirements.txt`.

## Dataset Setup

Download the UNSW-NB15 CSV files from the [official UNSW-NB15 project page](https://research.unsw.edu.au/projects/unsw-nb15-dataset) or the [Kaggle dataset mirror](https://www.kaggle.com/datasets/mrwellsdavid/unsw-nb15).

The experiment expects these files:

```text
UNSW_NB15_training-set.csv
UNSW_NB15_testing-set.csv
```

Place the files in the directory expected by the pipeline, or set `DATA_DIR` if supported by the current code.

**Linux or macOS**

```bash
export DATA_DIR="/path/to/dataset"
```

**Windows PowerShell**

```powershell
$env:DATA_DIR = "C:\path\to\dataset"
```

Check `nids_pipeline.py` for the exact file names, path resolution and environment-variable behaviour before running the experiment.

The pipeline's data-integrity checks should be allowed to complete. If a file hash differs from the expected manifest, investigate the source and version of the downloaded data rather than silently bypassing the check.

## Running the Pipeline

From the repository root, run:

```bash
python nids_pipeline.py
```

The reported pipeline covers preprocessing, internal splitting, model training, validation-based rule and fusion selection, baseline comparisons, ablation studies, multi-seed evaluation and duplicate-sensitivity analysis.

Runtime depends on hardware, data location, model settings and the number of experiments. Review the script's configuration and output paths before starting a full run.

For reproducibility:

1. Start from a clean environment.
2. Use the documented dataset files.
3. Record the code version, configuration and dependency versions.
4. Preserve the generated logs and result files.
5. Verify the final tables against the outputs of the exact code version used for the experiment.

Do not assume that saved model weights or CSV files correspond to the latest code unless they were regenerated or explicitly matched to that version.

## Reproducibility and Generated Artifacts

The experiment is designed to record data manifests, model configurations, split indices, validation searches and evaluation results.

For a reproducible release, make available:

- The source code and dependency specification.
- Dataset source and integrity manifest.
- Experiment configuration and random seeds.
- Per-seed results and paired ablation differences.
- Test-set predictions or scores where sharing is permitted.
- Logs needed to verify the main claims.
- Instructions for regenerating the tables from the saved outputs.

The official dataset files may be subject to their own access and distribution conditions. Users should obtain the dataset from its source rather than assume that this repository redistributes it.

## Limitations

The results should be interpreted in light of the following limitations:

- **Dataset scope:** The experiments use one benchmark with flow-level features. Generalisation to real network traffic has not been established.
- **Cross-split overlap:** Model-visible feature vectors occur in both official partitions. Internal group-aware splitting does not eliminate this official-split overlap.
- **Deduplication confounding:** The deduplicated test set has a different class prevalence and attack-category composition, preventing a simple causal interpretation of the resulting performance change.
- **Heuristic rule design:** Only three percentile-based rules are evaluated. The findings do not cover richer protocol invariants, expert systems or signature-based detection.
- **Seed variability:** The random-subspace ensemble shows substantial variation across the reported runs.
- **Calibration:** Fusion and threshold selection depend on the validation protocol. Results may change under different thresholds, data distributions or operating constraints.
- **Deployment conditions:** The benchmark's attack prevalence and evaluation setup do not establish a realistic production false-alarm rate or alert budget.
- **Supervised comparator:** The Random Forest uses labelled data and has a substantially different false-positive trade-off.

## Citation

If you use this code or experimental framework, cite the repository as a software artifact:

```bibtex
@misc{sattlapally2026neurosymbolic,
  author       = {Sattlapally, Viveka},
  title        = {Symbolic Rules Add No Measurable Benefit:
                  A Controlled Evaluation of Neuro-Symbolic
                  Network Intrusion Detection},
  year         = {2026},
  howpublished = {\url{https://github.com/vs2806m/COMP1818-NIDS-Project}},
  note         = {Research project repository}
}
```

Please also cite the original dataset and relevant methods papers when using their data or techniques. In particular, distinguish this Kitsune-inspired random-subspace ensemble from the original Kitsune/KitNET method.

## License

This project is distributed under the license specified in the repository's [`LICENSE`](LICENSE) file. Check that the file exists and contains the intended license text before describing the project as MIT-licensed.

## Acknowledgements

The author acknowledges the creators of UNSW-NB15 for making the dataset available for network intrusion detection research.
