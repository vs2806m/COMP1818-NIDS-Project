# Data

Place the two UNSW-NB15 CSV files here:

- `UNSW_NB15_training-set.csv`
- `UNSW_NB15_testing-set.csv`

Download them from the official source:
https://www.kaggle.com/datasets/mrwellsdavid/unsw-nb15

The CSVs are not committed to this repository (see `.gitignore`). Some
mirrors swap the two filenames; the pipeline detects this from the row
counts (175,341 training / 82,332 testing) and handles it automatically.
