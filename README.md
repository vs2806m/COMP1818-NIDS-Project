# Artifacts

This directory contains the outputs of the run reported in the paper,
produced by `nids_pipeline.py`. They are committed so a reader can inspect
the results and reload the trained models without re-running the pipeline.

## Results and configuration

| File | Produced by | Contents |
|---|---|---|
| `run_log.txt` | Whole run | Console output of the reported run |
| `final_experimental_results.csv` | HELD-OUT TEST EVALUATION | Per-model accuracy, macro-F1, MCC, ROC-AUC, AP, precision, recall, FPR and confusion-matrix counts on the official test set |
| `multi_seed_sensitivity.csv` | MULTI-SEED SENSITIVITY | Per-seed, per-model metrics across the five seeds |
| `multi_seed_paired_deltas.csv` | MULTI-SEED SENSITIVITY | Per-seed paired MCC differences (hybrid − zeroed, hybrid − control, hybrid − AE) |
| `unseen_indicator_ablation.csv` | MULTI-SEED SENSITIVITY | Per-seed MCC with and without the unseen-category indicators |
| `test_dedup_sensitivity.csv` | DEDUPLICATED-TEST SENSITIVITY | Metrics on the test set with within-test duplicates removed |
| `attack_category_results.csv` | ATTACK-CATEGORY BREAKDOWN | Per-category recall for each model |
| `rule_validation_search.csv` | SYMBOLIC RULES | Rule percentile/duration grid scored on `val_rules` |
| `fusion_validation_search.csv` | HYBRID FUSION | α, γ, threshold grid scored on `val_fusion` |
| `thresholds.json` | End of run | All selected thresholds and implied effective thresholds |
| `experiment_config.json` | End of run | Full configuration, loss histories, rule metadata |
| `baseline_config.json` | End of run | Isolation Forest and One-Class SVM metadata |
| `data_manifest.json` | End of run | Dataset hashes, split sizes, drop reasons, preprocessing policy |
| `feature_names.json` | End of run | Names of the 63 model input features |
| `requirements.txt` | End of run | Package versions of the environment that produced this run |

## Trained models and preprocessing

| File | Size | Contents |
|---|---:|---|
| `autoencoder_state_dict.pt` | ~25 KB | Trained deep autoencoder (PyTorch `state_dict`) |
| `ensemble_state_dict.pt` | ~31 KB | Trained random-subspace ensemble |
| `ensemble_subspace_indices.npy` | ~1 KB | Feature partition needed to rebuild the ensemble |
| `preprocessor.joblib` | ~6 KB | Fitted scaler and encoder for the unsupervised models |
| `rf_preprocessor.joblib` | ~7 KB | Fitted preprocessor for the Random Forest |
| `*_indices.npy` | ~1.4 MB total | Internal split and OCSVM subsample indices |

`rf_comparator.joblib` (~37 MB) is **not** committed. The Random Forest is a
supervised reference comparator, not the proposed model; regenerate it by
running the pipeline.

## How to regenerate

From the repository root, with the dataset CSVs in `data/`:

```bash
cd artifacts
DATA_DIR=../data python ../nids_pipeline.py | tee run_log.txt
```

The script writes every output to the current working directory, which is
why it is run from inside `artifacts/`. Seeds are fixed, so the run should
reproduce these files with the pinned package versions; small numerical
differences are possible across hardware or library versions.

A fresh rerun of the notebook on Colab reproduced every file in this folder
with identical sizes and identical reported metrics.

## Reloading the trained models

The saved preprocessors were created with **scikit-learn 1.6.1** and must be
loaded with that version (it is pinned in `requirements.txt`); newer
scikit-learn releases cannot unpickle them. Run from the repository root:

```python
import json, sys, numpy as np, torch, joblib
sys.path.insert(0, ".")
import nids_pipeline as P

preprocessor = joblib.load("artifacts/preprocessor.joblib")
thresholds = json.load(open("artifacts/thresholds.json"))

autoencoder = P.DeepAutoencoder(63, P.CONFIG["ae_hidden"], P.CONFIG["ae_latent"])
autoencoder.load_state_dict(torch.load("artifacts/autoencoder_state_dict.pt", map_location="cpu"))
autoencoder.eval()

ensemble = P.RandomSubspaceEnsemble(
    63, P.CONFIG["ensemble_preferred_subspaces"], P.CONFIG["ensemble_subspace_max"],
    P.CONFIG["ensemble_sub_hidden"], P.CONFIG["ensemble_sub_latent"],
    P.CONFIG["ensemble_out_hidden"], seed=42,
    subspace_indices=np.load("artifacts/ensemble_subspace_indices.npy", allow_pickle=True),
)
ensemble.load_state_dict(torch.load("artifacts/ensemble_state_dict.pt", map_location="cpu"))
ensemble.eval()
```

An autoencoder anomaly score is the per-row reconstruction MSE divided by
`thresholds["autoencoder_unsupervised"]`; a row is flagged when the score
exceeds 1.0. Using the saved preprocessor, models and thresholds on the test
set reproduces the reported MCCs: autoencoder 0.6259, label-tuned control
0.6395, hybrid 0.6374, rules-zeroed 0.6387 and ensemble 0.5568.
