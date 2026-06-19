# Context Document — Anomaly Detection Benchmark Pipeline
> Feed this file to any model to resume this project without loss of context.
> Last updated: June 2026 — **Data collection complete.**

---

## Current Status: ✅ COMPLETE

All 6,120 benchmark rows collected, verified, and analyzed.
The paper can now be written directly from `benchmark_report.md`.

---

## Project Overview

A **4-way comparative benchmark** on MVTec-AD (15 categories) measuring:
1. How PatchCore and PaDiM degrade under 5 corruption types (3 severities each)
2. Whether classical rescue preprocessing restores detection performance
3. Whether training-time corruption augmentation improves robustness
4. Whether combining augmented training + rescue gives additive benefit

**Core result**: Augmented training helps (+10–12 pp mean AUROC). Rescue preprocessing hurts (net-negative in all 4 conditions). Combining them is worst.

---

## Repository Location

```
/home/abdullah/Downloads/anomaly-detection/
```

---

## Experiment Design

### Models
- **PatchCore**: `backbone=wide_resnet50_2`, `num_neighbors=9`, `max_epochs=1`
- **PaDiM**: `backbone=wide_resnet50_2`, `layers=[layer1,layer2,layer3]`, `n_features=100`, `max_epochs=1`

### 4 Conditions
| Condition | Script | Output CSV |
|---|---|---|
| PatchCore — Clean | `run_patchcore.py` | `data/patchcore_clean_all_severities.csv` |
| PaDiM — Clean | `run_padim.py` | `data/padim_clean_all_severities.csv` |
| PatchCore — Augmented | `run_patchcore_augmented.py` | `patchcore_augmented.csv` |
| PaDiM — Augmented | `run_padim_augmented.py` | `padim_augmented.csv` |

### Row structure (34 rows per category/seed pair)
- 1 × baseline (clean test AUROC post-training)
- 5 ctypes × 3 severities = 15 degradation rows
- 6 rescue streams × 3 severities = 18 rescue rows

### Augmented training mechanism
`prepare_augmented_train_data(aug_prob=0.5)` pre-generates corrupted training images to disk before `engine.fit()`. Each training image is independently corrupted (50% chance, random type + severity). `MVTecAD(root=AUG_TRAIN_ROOT)` is then used for training — identical API to clean scripts.

---

## Key Technical Decisions & Fixes

### Critical Bug (RESOLVED): Silent no-op corruption
`datamodule.test_data = CorruptedDatasetWrapper(...)` was silently ignored by Anomalib.
`engine.test(datamodule=dm)` rebuilds from its own cached dataset.
**Fix**: Build DataLoader ourselves and pass via `engine.test(dataloaders=...)`.

### Environment hardening (Kaggle)
```python
os.environ["ANOMALIB_USE_RICH"] = "0"
os.environ["RICH_NO_THEME"] = "1"
sys.setrecursionlimit(5000)
```
Required to prevent Kaggle's rich/Jupyter recursion crash.

### Resume logic
```python
completed_keys = set()
for _, row in old_df.iterrows():
    completed_keys.add((row['category'], int(row['seed'])))
```
Checks `(category, seed)` pairs. If a pair is in the CSV, it's skipped.
**Important**: If a run times out mid-pair, delete those partial rows before re-running.

### CUDA compatibility
Install with `pip install --no-deps` for torch-dependent packages to avoid overwriting the Kaggle CUDA build.

---

## Data Files (Canonical)

| File | Rows | Content |
|---|---|---|
| `data/benchmark_full_4way.csv` | 6,120 | **All 4 conditions unified** |
| `data/benchmark_clean_all_severities.csv` | 3,060 | Clean training, both models |
| `patchcore_augmented.csv` | 1,530 | Augmented training, PatchCore |
| `padim_augmented.csv` | 1,530 | Augmented training, PaDiM |
| `data/patchcore_clean_all_severities.csv` | 1,530 | Clean training, PatchCore |
| `data/padim_clean_all_severities.csv` | 1,530 | Clean training, PaDiM |

## Analysis Files

| File | Content |
|---|---|
| `analyze_results.py` | Full 4-way analysis → 6 figures in `results/analysis/` |
| `benchmark_report.md` | Paper-ready report with all tables and findings |
| `results/analysis/*.png` | 6 publication figures |

---

## Confirmed Results

```
Rescue Success Rates (% of instances where rescue AUROC > deg AUROC):
  PatchCore — Clean       25.8%  mean Δ = -0.0717
  PatchCore — Augmented   22.7%  mean Δ = -0.1490   ← worst
  PaDiM     — Clean       35.1%  mean Δ = -0.0456
  PaDiM     — Augmented   19.4%  mean Δ = -0.1134   ← worst

Augmented Training Gains (mean degradation AUROC):
  PatchCore:  Clean=0.7356  Aug=0.8588  Δ=+0.1232
  PaDiM:      Clean=0.6390  Aug=0.7400  Δ=+0.1009

Baseline AUROC (clean test images, post-training):
  PatchCore — Clean:      0.9822 ± 0.0235
  PatchCore — Augmented:  0.9773 ± 0.0249  (cost: -0.5 pp)
  PaDiM     — Clean:      0.9160 ± 0.0801
  PaDiM     — Augmented:  0.8979 ± 0.0770  (cost: -2.5 pp)
```

---

## What's Next

1. **Statistical significance tests** — Wilcoxon signed-rank on rescue deltas and aug-training gains
2. **Figure polish** — colorblind palette, LaTeX labels (if IEEE/CVPR)
3. **VisA generalization** (optional) — needed for top-venue papers, script exists at `notebooks/09_visa_generalization.py`
4. **Write paper** — full structure, tables, and conclusions in `benchmark_report.md`
