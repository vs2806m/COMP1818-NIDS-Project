# Data

Place the two UNSW-NB15 CSV files here:

- `UNSW_NB15_training-set.csv`
- `UNSW_NB15_testing-set.csv`

Download them from the Kaggle copy used for this project:
https://www.kaggle.com/datasets/mrwellsdavid/unsw-nb15

Original source (UNSW Canberra):
https://research.unsw.edu.au/projects/unsw-nb15-dataset

The CSVs are not committed to this repository (see `.gitignore`). In the
Kaggle copy the two filenames are swapped relative to the official split;
the pipeline detects this from the row counts (175,341 training / 82,332
testing) and handles it automatically.
