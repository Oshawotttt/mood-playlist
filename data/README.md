# Data

Datasets aren't committed to git. Download them from the sources below and put the files here:

```
data/
├── raw/
│   ├── muse/muse_v3.csv
│   └── empatheticdialogues/{train,valid,test}.csv
└── interim/
    └── ed_situations.csv     # built from raw/empatheticdialogues: one row per situation
```

| Dataset | Download | Paper | Licence |
|---|---|---|---|
| MuSe | [Kaggle](https://www.kaggle.com/datasets/cakiki/muse-the-musical-sentiment-dataset) (Download button) | [Akiki & Burghardt, 2020](http://ceur-ws.org/Vol-2723/short26.pdf) | CC BY 4.0 |
| EmpatheticDialogues | [GitHub](https://github.com/facebookresearch/EmpatheticDialogues) (download link in the README) | [Rashkin et al., 2019](https://arxiv.org/abs/1811.00207) | CC BY-NC 4.0 (non-commercial, don't republish) |

Details and caveats: [../proposal/dataset_review.md](../proposal/dataset_review.md).
