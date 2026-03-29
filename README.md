# 🚴 cycling-entropy

R analysis pipeline for the study of **recurrence-matrix entropy during cycling and rest**, using scalp EEG recorded from healthy adults in four behavioural conditions.

[![PLOS ONE](https://img.shields.io/badge/PLOS%20ONE-10.1371%2Fjournal.pone.0298703-brightgreen)](https://doi.org/10.1371/journal.pone.0298703)
[![bioRxiv](https://img.shields.io/badge/bioRxiv-2024.01.31.578253-b31b1b)](https://doi.org/10.1101/2024.01.31.578253)
[![OSF](https://img.shields.io/badge/OSF-CW87P-blue)](https://doi.org/10.17605/OSF.IO/CW87P)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![R](https://img.shields.io/badge/R-≥4.3-276DC3.svg)](https://www.r-project.org/)

---

This repository contains all data and analysis code for:

> Ferré IBS, Corso G, dos Santos Lima GZ, Lopes SR, Leocadio-Miguel MA, França LGS, de Lima Prado T, Araújo JF (2024).
> Cycling reduces the entropy of neuronal activity in the human adult cortex.
> *PLOS ONE*, 19(10): e0298703. https://doi.org/10.1371/journal.pone.0298703

The entropy estimation script used to produce the `.dat` files in `data/` is available at
[circadia-bio/maxEntropy](https://github.com/circadia-bio/maxEntropy).
Raw EEG and entropy data are also archived on OSF at https://doi.org/10.17605/OSF.IO/CW87P.

---

## 📖 Study overview

**24 healthy adults** (13 women, 11 men; mean age ~21 years) performed two 2-minute conditions — resting and cycling on a horizontal stationary bicycle — with both open and closed eyes, yielding four behavioural states per participant. EEG was recorded from eight bimodal electrode pairs covering frontal, central, parietal, and occipital scalp regions. Recurrence-matrix entropy (S_Max) was computed for each 300-sample window across the recording, producing 100 time points per condition per electrode per participant.

Key findings: cycling was associated with significantly reduced cortical entropy across all eight electrode sites compared to rest. A further entropy reduction was observed when cycling with eyes closed, most prominently at right occipital and left parietal sites.

---

## 🗂️ Repository structure

```
cycling-entropy/
├── data/                    # Pre-computed entropy data (see Data dictionary)
│   ├── SMax_<col>_col_<cond>.dat   # Entropy time series, one file per electrode × condition
│   ├── openPedalling.csv    # Median entropy per electrode, cycling eyes open
│   ├── openResting.csv      # Median entropy per electrode, resting eyes open
│   ├── closedPedalling.csv  # Median entropy per electrode, cycling eyes closed
│   └── closedResting.csv    # Median entropy per electrode, resting eyes closed
├── src/
│   └── legacy.R             # Data loading and reshaping (sourced by translating.Rmd)
├── translating.Rmd          # Main analysis notebook (GLMMs, figures 1–3)
├── run.R                    # One-line render script
├── Fig2.R                   # Standalone script for Figure 2 (eyes open vs closed)
├── Fig3.R                   # Standalone script for Figure 3 (cycling vs resting)
├── Fig4.R                   # Standalone script for Figure 4 (interaction: state × eyes)
├── topoplots.Rmd            # Topographic plot generation (Figure 3 of the paper)
├── rstudio/                 # Docker image definition for RStudio Server
├── jupyter/                 # Docker image definition for Jupyter (Python utilities)
├── docker-compose.yml       # Launches RStudio + Jupyter environments
└── cycling-entropy.Rproj   # RStudio project file
```

---

## 🗃️ Data dictionary

### `data/SMax_<col>_col_<cond>.dat`

The primary entropy data. Each file is a plain-text space-delimited matrix with **100 rows × 18 columns**, where:

- **Rows** = time windows (100 windows of 300 samples each, from the 2-minute recording)
- **Columns** = participants (18 subjects after exclusions; see IDs below)

**File naming convention:**

```
SMax_<col>_col_<cond>.dat
```

| Component | Values | Description |
|-----------|--------|-------------|
| `<col>` | `2`–`9` | Column index from the original Julia output, mapping to an electrode (see table below) |
| `<cond>` | `POA`, `POF`, `ROA`, `ROF` | Condition code (see table below) |

**Condition codes:**

| Code | Movement | Eyes | Label in analysis |
|------|----------|------|-------------------|
| `POA` | **P**edalling | **O**pen (**A**berto) | Pedalling / Open |
| `POF` | **P**edalling | **F**echado (Closed) | Pedalling / Closed |
| `ROA` | **R**esting | **O**pen (**A**berto) | Resting / Open |
| `ROF` | **R**esting | **F**echado (Closed) | Resting / Closed |

**Column-to-electrode mapping:**

| `<col>` | Electrode pair | Brain region | Hemisphere | 10-20 label |
|---------|---------------|--------------|------------|-------------|
| `2` | O2-A1 | Occipital | Right | O2 |
| `3` | O1-A2 | Occipital | Left | O1 |
| `4` | P4-Pz | Parietal | Right | P4 |
| `5` | P3-Pz | Parietal | Left | P3 |
| `6` | C4-Cz | Central | Right | C4 |
| `7` | C3-Cz | Central | Left | C3 |
| `8` | F4-Fz | Frontal | Right | F4 |
| `9` | F3-Fz | Frontal | Left | F3 |

**Participant IDs (columns 1–18):**

```
I1, I2, I3, I4, I5, I6, I7, I8, I12, I13, I14, I15, I17, I18, I20, I21, I22, I24
```

IDs are non-consecutive because participants I9, I10, I11, I16, I19, and I23 were excluded prior to analysis (EEG artefacts from poor electrode contact; see paper Methods).

**Quality control:** In `src/legacy.R`, any participant whose entropy values for a given electrode have a standard deviation > 10 across the four conditions is set to `NA` for that electrode. This threshold (`sd_cri = 10`) is defined at the top of the script.

---

### `data/{open,closed}{Pedalling,Resting}.csv`

Summarised median entropy per electrode, used for the topographic plots (Figure 3 of the paper). Each file has two columns:

| Column | Description |
|--------|-------------|
| `electrode` | Standard 10-20 label: `F3`, `F4`, `C3`, `C4`, `P3`, `P4`, `O1`, `O2` |
| `amplitude` | Median S_Max across participants and time windows for that condition |

These are derived from the long-format `df` object in `translating.Rmd` via `topoplots.Rmd`.

---

## 🔬 Analysis pipeline

### `src/legacy.R`

Sourced at the start of `translating.Rmd`. Reads all 32 `.dat` files (8 electrodes × 4 conditions), reshapes each into a long-format data frame with columns `ind`, `olho`, `MOV`, `POS`, `LAT`, `dados`, applies the SD-based quality-control filter, and combines everything into a single tibble `pla`.

**Variable names in `pla` / `df` after recoding:**

| Variable | Type | Values | Description |
|----------|------|--------|-------------|
| `sub` | factor | `I1`–`I24` | Participant ID |
| `State` | factor | `Pedalling`, `Resting` | Movement condition; reference level = `Resting` |
| `Eyes` | factor | `Open`, `Closed` | Eye condition; reference level = `Open` |
| `Electrode` | factor | `Central`, `Frontal`, `Occipital`, `Parietal` | Brain region |
| `Side` | factor | `Left`, `Right` | Hemisphere |
| `value` | numeric | ~3–6 | S_Max entropy estimate for the time window |

### `translating.Rmd`

The main analysis notebook. Renders to a self-contained HTML report. Key sections:

**GLM1 — eyes open vs closed (validation):**
```r
med_entr ~ Eyes + (1|sub)
```
Fit per electrode × side, tested with 10 000-repetition two-sided permutation tests. Confirms the expected entropy reduction with eyes closed in the occipital region (t = −2.657, p = 0.015), validating the method against established alpha-rhythm physiology.

**GLMM2 — cycling vs resting (main effect):**
```r
med_entr ~ State + (1|sub)
```
Significant entropy reduction with cycling across all eight electrode sites (all p ≤ 0.042).

**GLMM2 — interaction (State × Eyes):**
```r
med_entr ~ State * Eyes + (1|sub)
```
Additional entropy reduction for cycling with eyes closed at right occipital (p = 0.045) and left parietal (p = 0.006).

All models use `grouped_perm_glmm()` from the `ptestr` package with `seed = 42` and `nPerms = 10000` for reproducibility.

### `Fig2.R`, `Fig3.R`, `Fig4.R`

Standalone R scripts that reproduce the three main figures. Each loads `legacy.R` directly and produces a PDF. `Fig2.R` also contains an earlier base-R boxplot version (`fig2.pdf`) used during development.

### `topoplots.Rmd`

Generates the topographic scalp maps (Figure 3 in the paper) using a GAM-based spatial interpolation approach (`mgcv`, `akima`) applied to the four summarised CSV files. Electrode positions follow the standard 10-20 system.

---

## 🚀 Getting started

### Option A — Docker (recommended, fully reproducible)

Requires [Docker](https://www.docker.com/) and Docker Compose.

```bash
git clone https://github.com/circadia-bio/cycling-entropy.git
cd cycling-entropy
docker-compose up
```

This launches two services:

| Service | URL | Description |
|---------|-----|-------------|
| RStudio Server | http://localhost:8789 | Full R environment with all packages pre-installed; password: `letmein` |
| JupyterLab | http://localhost:8888 | Python environment for data inspection utilities |

Open RStudio, navigate to `cycling-entropy/`, and open `translating.Rmd` or run `source('run.R')` to render the full report.

### Option B — Local R

Requires R ≥ 4.3 and the following packages:

```r
install.packages(c(
  "tidyverse",
  "ggbeeswarm",
  "ggpubr",
  "rstatix",
  "eegUtils",
  "rmarkdown",
  "akima",
  "scales",
  "mgcv",
  "gridExtra",
  "png"
))

# ptestr is not on CRAN — install from GitHub:
remotes::install_github("quantide/ptestr")
```

Then render the full report:

```r
rmarkdown::render("translating.Rmd", "html_document")
```

Or run individual figure scripts:

```r
source("Fig2.R")
source("Fig3.R")
source("Fig4.R")
```

---

## ⚙️ Reproducing from raw entropy files

The `.dat` files in `data/` are the output of [`circadia-bio/maxEntropy`](https://github.com/circadia-bio/maxEntropy), which computes recurrence-matrix entropy (S_Max) from the raw EEG recordings. If you have access to the original EEG files (available on [OSF](https://doi.org/10.17605/OSF.IO/CW87P)), you can regenerate the `.dat` files by running `Entropy.jl` from the maxEntropy repository against each participant's recording.

The full data provenance chain is:

```
Raw EEG (.txt, PowerLab/LabChart)
    ↓  Entropy.jl  (circadia-bio/maxEntropy)
data/SMax_<col>_col_<cond>.dat
    ↓  src/legacy.R  →  translating.Rmd
Figures 1–3  +  statistical results
```

---

## 📦 R package dependencies

| Package | Purpose |
|---------|---------|
| `tidyverse` | Data wrangling and `ggplot2` figures |
| `ggbeeswarm` | Quasirandom jitter overlays on boxplots |
| `ggpubr` | Statistical annotation on ggplot figures |
| `rstatix` | Paired t-tests and effect sizes |
| `ptestr` | Permutation-based GLMM testing (`grouped_perm_glmm`) |
| `eegUtils` | EEG-specific utilities and topoplot support |
| `akima` | 2D interpolation for topographic maps |
| `mgcv` | GAM-based spatial smoothing |
| `scales` | Axis and colour scale helpers |
| `gridExtra` | Multi-panel figure layout |
| `rmarkdown` | Rendering `.Rmd` to HTML |

---

## 📄 Citation

```bibtex
@article{ferre2024cycling,
  title   = {Cycling reduces the entropy of neuronal activity in the human adult cortex},
  author  = {Ferr{\'e}, Iara Beatriz Silva and Corso, Gilberto and
             dos Santos Lima, Gustavo Zampier and Lopes, Sergio Roberto and
             Leocadio-Miguel, Mario Andr{\'e} and Fran{\c{c}}a, Lucas G S and
             de Lima Prado, Thiago and Ara{\'u}jo, John Fontenele},
  journal = {PLOS ONE},
  volume  = {19},
  number  = {10},
  pages   = {e0298703},
  year    = {2024},
  doi     = {10.1371/journal.pone.0298703}
}
```

---

## 🤝 Related tools

- 🔄 [**maxEntropy**](https://github.com/circadia-bio/maxEntropy) — Julia script that computed the S_Max entropy values in `data/`
- 🌙 [**SleepDiaries**](https://github.com/circadia-bio/SleepDiaries) — open-source React Native sleep diary app
- ⚡ [**ACTT_validation_study**](https://github.com/circadia-bio/ACTT_validation_study) — ActTrust® validation pipeline in R/Quarto
- 🔬 [**circadia-bio**](https://github.com/circadia-bio) — the Circadia Lab GitHub organisation

---

## 📄 Licence

[MIT](LICENSE) © 2024 circadia-bio
