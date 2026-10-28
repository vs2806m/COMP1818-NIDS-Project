markdown
<div align="center">

# Neuro-Symbolic Network Anomaly Detection on UNSW-NB15

### Integrating Autoencoder-Based Detection with Knowledge-Based Rules

[![Python](https://img.shields.io/badge/Python-3.13-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![PyTorch](https://img.shields.io/badge/Framework-PyTorch-EE4C2C?logo=pytorch&logoColor=white)](https://pytorch.org/)
[![Dataset](https://img.shields.io/badge/Dataset-UNSW--NB15-2563EB)](https://research.unsw.edu.au/projects/unsw-nb15-dataset)
[![Course](https://img.shields.io/badge/Course-COMP1818-6D28D9)](#)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

**University of Greenwich · COMP1818 Artificial Intelligence Applications**

*A controlled evaluation of whether simple symbolic rules add measurable value to autoencoder-based network anomaly detection.*

</div>

---

## Overview

This repository contains the implementation and experimental pipeline for
evaluating whether a small set of heuristic symbolic rules improves
unsupervised network anomaly detection on the **UNSW-NB15** dataset.

The study compares classical unsupervised detectors, a deep autoencoder,
a Kitsune-inspired random-subspace ensemble, three percentile-based
symbolic rules, and a hybrid score-fusion approach. Matched ablations
and multi-seed experiments investigate the effects of rule activation,
threshold calibration, duplicate records, and random-seed variation.

> **Main finding**
>
> In the reported single-seed experiment, the hybrid detector achieved an
> MCC of **0.6374**, compared with **0.6387** for the rules-zeroed control.
> Across five seeds, the reported paired difference was
> **+0.0004 ± 0.0028**. These results do **not** demonstrate a robust
> performance advantage from adding the symbolic rules.

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

**Do manually specified percentile-based symbolic rules improve the
performance of an autoencoder-based network intrusion detection system on
UNSW-NB15, and how robust are the observed results to threshold selection,
duplicate records, and random-seed variation?**

The evaluation focuses on detection quality, false-positive behaviour, and
the consistency of observed performance differences across experimental
conditions.

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

The hybrid detector performs slightly better than the baseline autoencoder
in the reported primary run, but it does not outperform the rules-zeroed
control. The multi-seed difference is small and does not establish a
robust advantage attributable to symbolic-rule integration.

The Random Forest is included as a **supervised reference comparator**,
not as a directly equivalent unsupervised baseline, because it uses
labelled training data.

---

## Methodology

### 1. Dataset and preprocessing

The project uses the official UNSW-NB15 training and testing partitions.

| Partition | Records |
|---|---:|
| Training | 175,341 |
| Testing | 82,332 |

Preprocessing includes:

- Excluding identifiers and selected features, including attack-category
  labels, TCP sequence numbers, and selected TTL-related features.
- Applying log transformations to selected heavy-tailed volume features.
- Fitting the scaler and one-hot encoder using normal training rows only.
- Creating three binary indicators for categorical values unseen in
  normal training traffic.
- Producing a final model input dimension of **63 features**.

Feature exclusions are experimental design choices; they should not be
interpreted as proof that the excluded features always constitute leakage
or testbed artefacts.

### 2. Internal data splits

The reported pipeline uses `GroupShuffleSplit` with groups derived from
SHA-256 hashes of model-visible feature vectors. The internal subsets are
`train`, `val_es`, `val_rules`, and `val_fusion`. The intention is to
prevent identical model-visible feature vectors from crossing these
internal subsets.

> **Data-split caveat:** These internal splits do not remove the reported
> overlaps between the official training and testing partitions. The
> primary benchmark should therefore not be described as completely
> leakage-free.

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

```text
63 → 32 → 16 → 32 → 63
ReLU hidden-layer activations, linear output, mean-squared-error loss,
Adam optimiser. Trained on normal traffic only.

The anomaly score is reconstruction MSE normalised by the 95th-percentile
reconstruction MSE of normal validation traffic.

Random-subspace ensemble
Eight small autoencoders and an output autoencoder. Inspired by
Kitsune/KitNET, but using random feature partitioning rather than
KitNET's correlation-based feature-mapping procedure. It should therefore
be regarded as Kitsune-inspired, not as a faithful reproduction of the
original system.

4. Symbolic rules
Three heuristic percentile-based rules flag potentially suspicious
traffic:

Flood-like behaviour.

Asymmetric outbound traffic.

Scan-like behaviour.

The rules are combined using logical OR:

S
S
y
m
=
R
1
∨
R
2
∨
R
3
S 
Sym
​
 =R 
1
​
 ∨R 
2
​
 ∨R 
3
​
 
The rules are selected using a validation procedure with a target
false-positive-rate cap of 2%. The reported fallback configuration does
not achieve that cap, so the target should not be interpreted as a
guaranteed operating point. These conditions are experiment-specific
heuristics, not a comprehensive expert knowledge base.

5. Hybrid score fusion
S
H
y
b
r
i
d
=
α
S
M
L
+
(
1
−
α
)
S
S
y
m
+
γ
(
S
M
L
⋅
S
S
y
m
)
S 
Hybrid
​
 =αS 
ML
​
 +(1−α)S 
Sym
​
 +γ(S 
ML
​
 ⋅S 
Sym
​
 )
where 
S
M
L
S 
ML
​
  is the autoencoder anomaly score,
S
S
y
m
S 
Sym
​
  is the binary symbolic-rule indicator, 
α
α
weights the machine-learning score, and 
γ
γ controls the interaction
term.

The reported configuration uses 
α
=
0.6
α=0.6, 
γ
=
0.1
γ=0.1,
t
=
1.0
t=1.0. When no symbolic rule fires, the fusion score becomes
0.6
 
S
M
L
0.6S 
ML
​
 , so under 
t
=
1.0
t=1.0 the decision rule
reduces to 
S
M
L
>
1
/
0.6
≈
1.667
S 
ML
​
 >1/0.6≈1.667. This algebra shows
that the hybrid decision rule changes the effective threshold when no rule
fires. It does not, by itself, prove that all observed performance
differences are caused by threshold calibration.

6. Evaluation design
Held-out test-set metrics.

Five random seeds: 42, 7, 123, 2024, 99.

Attack-category recall.

Duplicate-record sensitivity analysis.

Rules-zeroed ablation.

Label-tuned threshold control.

Logical OR and logical AND comparisons.

The label-tuned control uses validation labels (not test labels) for
threshold selection. It should not be interpreted as a deployable
unsupervised model.

Experimental results
Test set: 82,332 records, with reported attack prevalence of 55.1%.

Model	MCC	FPR
Isolation Forest	0.2907	0.0702
One-Class SVM	0.6125	0.0648
Symbolic rules only	0.2039	0.0274
Deep autoencoder	0.6259	0.0915
AE label-tuned control	0.6395	0.0672
Random-subspace ensemble	0.5568	0.0788
Logical OR	0.6077	0.1161
Logical AND	0.2515	0.0028
Rules-zeroed ablation	0.6387	0.0627
Hybrid detector	0.6374	0.0645
Supervised Random Forest	0.7506	0.2743
MCC = Matthews correlation coefficient; FPR = false-positive rate.

Interpretation
The hybrid detector improves on the baseline autoencoder in this single
run, but it does not improve on the rules-zeroed control or the
label-tuned threshold control. Across five seeds, the reported paired
difference between the hybrid and rules-zeroed control is small. The
results do not establish a robust advantage attributable to symbolic-rule
integration.

The Random Forest achieves a higher MCC but has a substantially higher
FPR. Its supervised training setup also differs from the unsupervised
approaches.

Dataset and data quality
Dataset source
Official source: UNSW-NB15 dataset page

Alternative mirror: UNSW-NB15 on Kaggle

Prefer the official source where possible. Check the dataset's usage
conditions and citation requirements before redistribution or commercial
use.

Duplicate-record audit
Audit	Reported count
Within-training duplicates (model-visible)	74,303
Within-testing duplicates (model-visible)	28,386
Cross-split overlaps (model-visible)	1,305
The project retains the official split for its primary benchmark and
conducts deduplication as a sensitivity analysis. Deduplication changes
the number of evaluated records and can change class prevalence and
attack-category composition. Performance reductions after deduplication
therefore cannot automatically be attributed solely to duplicate removal.

The reported overlap between the official training and testing partitions
means the primary benchmark should not be described as entirely free from
cross-split overlap. The group-aware internal splits address a separate
issue: repeated model-visible feature vectors crossing the internal
validation subsets.
