# Implementation Plan: Monolithic Anomaly Detection Benchmark

This document outlines the finalized architecture and execution strategy for the Robust Industrial Anomaly Detection benchmark.

## Project Decisions & Architecture Shift

### 1. Monolithic Script Structure
To minimize upload friction and dependency issues on Kaggle, the project has moved from a multi-file `src/` directory to three standalone, high-performance Python scripts:
- **`03_severity_calibration.py`**: The Master Engine. Contains all configuration, corruption functions, and rescue algorithms. Generates visual grids and the `experiment_config.json`.
- **`run_patchcore.py`**: The PatchCore runner. Ingests config, executes the benchmark (Baseline -> Degradation -> Rescue), and saves results to CSV.
- **`run_padim.py`**: The PaDiM runner. Mirror logic of the PatchCore runner.

### 2. Finalized Corruption Configuration
Corruptions are applied on-the-fly using the following parameters, calibrated for perceptual proportionality:

| Corruption | Mild | Moderate | Severe |
| :--- | :--- | :--- | :--- |
| **Low Light** | gamma=0.5 | gamma=0.35 | gamma=0.2 |
| **Gaussian Blur** | sigma=5.0, k=31 | sigma=15.0, k=101 | sigma=25.0, k=281 |
| **Motion Blur** | k=31 | k=81 | k=151 |
| **Sensor Noise** | var=0.05 | var=0.18 | var=0.35 |
| **Fog/Haze** | density [0.2, 0.35] | density [0.45, 0.65] | density [0.75, 0.9] |

> [!NOTE]
> The **Combined** corruption has been removed from the final experiment suite to maintain focus on isolated physical corruptions and clearly attributable rescue performance.

### 3. Optimized Preprocessing (Rescue) Pipeline
Based on calibration results, the rescue pipeline has been pruned to only include high-impact algorithms:

- **Low Light**: `CLAHE` (Local Contrast) and `Retinex` (Illumination Normalization).
- **Gaussian Blur**: `Wiener Deconvolution` (using exact Gaussian PSF).
- **Motion Blur**: `Wiener Deconvolution` (using a specialized **Linear Motion PSF**).
- **Sensor Noise**: `NLM Denoise`. (BM3D removed due to prohibitive runtime/benefit ratio).
- **Fog/Haze**: `DCP Dehaze` (Dark Channel Prior).

> [!IMPORTANT]
> **Unsharp Masking** was removed as it provided negligible restoration compared to principled deconvolution.

## Execution Strategy (Kaggle-Optimized)

### In-Memory Loop
The benchmark follows a "Train Once, Test All" logic per category/seed:
1. Initialize Model (PatchCore or PaDiM).
2. Train on clean "good" images.
3. **Phase 1 (Baseline)**: Test on clean test images.
4. **Phase 2 (Degradation)**: Iterate through 5 corruptions × 3 severities.
5. **Phase 3 (Rescue)**: For every 'severe' corruption, apply valid preprocessing mappings.
6. Record results to `results/results.csv` and clear GPU memory.

### Determinism & Repeatability
- Stochastic corruptions (Noise, Fog) use a `seed` derived from the dataset item index: `seed = base_seed + index`.
- All runs are pinned to `seeds = [42, 123, 456]`.

## Deployment Checklist
1. **Run Calibration**: Execute `python3 notebooks/03_severity_calibration.py` to lock parameters and generate `experiment_config.json`.
2. **Execute PatchCore**: Run `python3 run_patchcore.py`.
3. **Execute PaDiM**: Run `python3 run_padim.py`.
4. **Collect Results**: Results are stored in the `results/` directory as CSV files.

---
*Last Updated: May 2026*
