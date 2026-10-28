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
