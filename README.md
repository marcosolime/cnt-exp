# CT Analysis with SaxNeRF

## Overview

This repository documents 3D reconstruction experiments on multi-wavelength CT datasets from Kubota et al. The datasets contain compositions of materials with varying densities (polymer, metal, etc.), scanned at **3 different wavelengths** to produce distinct absorption projections.

The detector used is a **Carbon Nano Tube (CNT) detector**, capable of detecting wavelengths across an ultra-wide range. For full details on the acquisition setup, refer to the Kubota et al. paper.

The goal is **volumetric 3D reconstruction**. While the original paper used shape-from-silhouette and visual hull reconstruction, this project explores [SaxNeRF](https://github.com/caiyuanhao1998/SAX-NeRF) as an alternative.

---

## Dataset

Raw datasets are located at:
```
/home/yixin/Desktop/NII佐藤先生-20251107T091146Z-1-001
```
> The source computer is at **desk 7, room 1612**.

The detector has a resolution of **3 × 20 pixels** (height=3, width=20).

Inside `（久保田君）CTイメージングの多波長データ/1,310 nm`, there are two input files:

| File | Description |
|------|-------------|
| `onoff.csv` | Per-pixel detector intensity recorded over ~100 timesteps (illuminated → dark) |
| `CTdata.csv` | Projection data, one row per acquisition angle. Each row contains 60 comma-separated values to be reshaped into a **3×20** matrix |

---

## Data Preprocessing

The preprocessing pipeline is defined in `calibration_1310.ipynb`. It converts raw string/CSV data into calibrated projection intensities.

Key steps:
1. Parse and calibrate raw intensities from `onoff.csv` and `CTdata.csv`
2. **Scale down projections by a factor of ~0.07** (empirically determined for SaxNeRF convergence)
3. Export projections to pickle format: `cnt_360.pickle`

---

## Training

The SaxNeRF repository is located on **NII server 2201NII**:
```
/home/yixin/workspace/SAX-NeRF
```

**Setup:**
1. Place the generated `cnt_360.pickle` inside the `data/` folder
2. Create a training config at `config/Lineformer/cnt_360.yaml`

**Start training:**
```bash
train_mlg --config config/Lineformer/cnt_360.yaml --gpu_id 0
```

**Monitor training:**
```
logs/Lineformer/
```

---

## Results

Full results are documented in the `CNT_Experiments` presentations.

### Reconstruction from CNT Projections

SaxNeRF **fails to converge** on the original Kubota-san dataset. The primary cause is the very low vertical resolution of the projections (only **3 pixels** tall). Approaches such as repeating or stretching the volume along the vertical axis did not yield meaningful improvements.

### Toy Volume Experiments

To isolate the failure mode, a synthetic toy volume was created — a **cuboid with a square hole** — and projections were generated using the [TIGRE toolkit](https://github.com/CERN/TIGRE) (installed inside the SaxNeRF repo).

**Pipeline for generating projections from a 3D voxel volume:**

```
volume.npy  ──►  dataGenerator/raw_data/cuboid/volume.npy
                 dataGenerator/raw_data/cuboid/config.yml   (projection settings)
                         │
                         ▼
              data_vis_cuboid.py       (intermediate .mat file + visualizations)
                         │
                         ▼
              generateData_cuboid.py   (outputs pickle to data/)
                         │
                         ▼
                   data/cuboid.pickle  ──►  SaxNeRF
```

### Convergence Limits

Through systematic experiments with the toy volume, the following thresholds were identified:

- ✅ SaxNeRF can reconstruct volumes from as **few as ~10 projections**
- ⚠️ A **minimum of ~20 pixels on both spatial dimensions** is required for a decent reconstruction
- ❌ The original CNT dataset (3-pixel vertical resolution) falls below this threshold

---

## Repository Structure

```
SAX-NeRF/
├── config/
│   └── Lineformer/
│       └── cnt_360.yaml          # Training hyperparameters
├── data/
│   └── cnt_360.pickle            # Preprocessed projections
├── dataGenerator/
│   ├── raw_data/cuboid/
│   │   ├── volume.npy
│   │   └── config.yml
│   ├── data_vis_cuboid.py
│   └── generateData_cuboid.py
└── logs/
    └── Lineformer/               # Training logs
```

---

## References

- Kubota et al. — *Multi-wavelength CT imaging with CNT detector* (see paper for full citation)
- [SAX-NeRF](https://github.com/caiyuanhao1998/SAX-NeRF) — Sparse-view CT reconstruction with NeRF
- [TIGRE Toolkit](https://github.com/CERN/TIGRE) — Tomographic iterative GPU-based reconstruction
