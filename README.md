# mood-playlist

**DAP EDA Notebooks:** [proposal/DAP_IEDA_Template.ipynb](proposal/DAP_IEDA_Template.ipynb) (main EDA) and [proposal/FAILED_edas.ipynb](proposal/FAILED_edas.ipynb) (failed Last.fm tag fetch).
## Data

Datasets aren't committed to git. Download them from the sources below into `data/raw/`.

- **MuSe**: download from [Kaggle](https://www.kaggle.com/datasets/cakiki/muse-the-musical-sentiment-dataset). Paper: [Akiki & Burghardt, 2020](http://ceur-ws.org/Vol-2723/short26.pdf). Licence: CC BY 4.0.
- **EmpatheticDialogues**: download from [GitHub](https://github.com/facebookresearch/EmpatheticDialogues). Paper: [Rashkin et al., 2019](https://arxiv.org/abs/1811.00207). Licence: CC BY-NC 4.0 (non-commercial, don't republish).
- **LRCLIB** (lyrics): no download. The EDA notebook fetches lyrics from the [LRCLIB API](https://lrclib.net) (free, no key) and caches them in `data/raw/lrclib/`. Lyrics are copyrighted: never commit or print them.

## EDA

[proposal/DAP_IEDA_Template.ipynb](proposal/DAP_IEDA_Template.ipynb) runs top to bottom from `proposal/`. It needs `pandas`, `numpy`, `matplotlib` and `langdetect`. The failed Last.fm tag fetch is kept in [proposal/FAILED_edas.ipynb](proposal/FAILED_edas.ipynb).
