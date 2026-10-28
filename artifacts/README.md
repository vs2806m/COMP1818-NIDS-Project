# Artifacts

This directory contains the small numerical outputs produced by
`nids_pipeline.py`. They are committed so a reader can inspect the reported
results without running the full pipeline (~30–45 minutes on CPU).

## What's here

| File | Produced by | Contents |
|---|---|---|
| `final_experimental_results.csv` | Section "HELD-OUT TEST EVALUATION" | Per-model accuracy, macro-F1, MCC, ROC-AUC, AP, precision, recall, FPR, confusion-matrix counts on the official test set |
| `multi_seed_sensitivity.csv` | Section "MULTI-SEED SENSITIVITY" | Per-seed, per-model metrics across the five sensitivity seeds |
| `multi_seed_paired_deltas.csv` | Section "MULTI-SEED SENSITIVITY" | Per-seed paired MCC differences (hybrid − zeroed, hybrid − control, hybrid − AE) |
| `unseen_indicator_ablation.csv` | Section "MULTI-SEED SENSITIVITY" | Per-seed MCC with and without the unseen-category indicators |
| `test_dedup_sensitivity.csv` | Section "DEDUPLICATED-TEST SENSITIVITY" | Metrics on the model-visible deduplicated test set |
| `attack_category_results.csv` | Section "ATTACK-CATEGORY BREAKDOWN" | Per-category recall for each model |
| `rule_validation_search.csv` | Section "SYMBOLIC RULES" | Grid of rule percentile / duration configurations scored on val_rules |
| `fusion_validation_search.csv` | Section "HYBRID FUSION" | Grid of α, γ, threshold configurations scored on val_fusion |
| `thresholds.json` | End of pipeline | All selected thresholds in one file |
| `experiment_config.json` | End of pipeline | Full configuration, loss histories, rule metadata |
| `baseline_config.json` | End of pipeline | Isolation Forest and One-Class SVM metadata |
| `data_manifest.json` | End of pipeline | Dataset hashes, split sizes, drop reasons, preprocessing policy |

Not committed (regenerate by running the pipeline):

- `preprocessor.joblib`, `rf_preprocessor.joblib`
- `autoencoder_state_dict.pt`, `ensemble_state_dict.pt`
- `rf_comparator.joblib`
- `*_indices.npy`
- `ocsvm_subsample_*.npy`, `ensemble_subspace_indices.npy`

These are large (up to ~37 MB) and are fully deterministic given the seed
and configuration, so they are excluded by `.gitignore` to keep the
repository lightweight.

## How to regenerate

From the repository root, with the dataset in place:

```bash
python nids_pipeline.py
