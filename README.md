# Data

Place the two UNSW-NB15 CSV files here:

- `UNSW_NB15_training-set.csv`
- `UNSW_NB15_testing-set.csv`

Download them from Kaggle:
https://www.kaggle.com/datasets/mrwellsdavid/unsw-nb15

The CSVs are not committed to this repository (see `.gitignore`). In the
Kaggle copy the two filenames are swapped relative to the official split
(`UNSW_NB15_training-set.csv` has 82,332 rows); the pipeline detects this
from the row counts (175,341 training / 82,332 testing) and handles it
automatically.

Expected SHA-256 hashes:

| File | Rows | SHA-256 |
|---|---:|---|
| `UNSW_NB15_testing-set.csv` | 175,341 | `bec7dd5ec88dc2a0ccc7a07879d338395ed7421750f675fd0339e07dfe0648fa` |
| `UNSW_NB15_training-set.csv` | 82,332 | `734fe6642edf758f7c94d7d9149426b49d202fe8e7bf0bef47392489c3c0a559` |
