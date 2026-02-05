<img width="740" height="738" alt="image" src="https://github.com/user-attachments/assets/e18b80a8-cc63-407d-a6ab-7d1d44f95300" />

<img width="717" height="955" alt="image" src="https://github.com/user-attachments/assets/0f809102-a0e2-47c4-8099-28f3847ec449" />



[README.md](https://github.com/user-attachments/files/25089411/README.md)
# GeoTIFF Data Inspection Toolkit

A Jupyter-based toolkit for inspecting satellite imagery stored as GeoTIFF files.
Built for **flood mapping** and **water body detection** research workflows, but works with any multi-band GeoTIFF data.

Designed to handle diverse dataset structures — from a single `.tif` file to deeply nested ROI directories — without forcing everything into one rigid UI.

---

## Features

### Automatic Data-Type Detection

The toolkit infers what kind of data you're looking at using a two-stage heuristic:

1. **Filename keywords** (highest priority) — e.g. `*S2*` → Sentinel-2 optical, `*sar*` → SAR, `*label*` → water mask
2. **Band count + value range** (fallback) — e.g. 13 bands → S2, 2 bands with negative dB → S1 SAR, 1 band with ≤12 unique values → label/DW

Supported data types:

| Type | Bands | Detection Keywords |
|------|-------|--------------------|
| Sentinel-2 TOA | 13 | `s2`, `optical`, `toa`, `sr` |
| Sentinel-1 SAR | 2 | `s1`, `sar` |
| Cloud/Shadow Mask | 5 | `cld`, `shdw`, `cloud` |
| Dynamic World | 1 | `dw`, `dynamic` |
| Water Label | 1 | `label`, `mask`, `water`, `jrc` |

### Visualization Modes

**`single`** — Single band image with histogram and cumulative distribution function (CDF).
Prints per-band statistics: min, max, mean, std, median, NaN%.

**`all`** — Grid view of every band with percentile-stretched display.
Automatically generates an inter-band **Pearson correlation heatmap** when there are 2+ bands.

**`rgb`** — Sensor-aware composite generation:
- *Optical*: True Color (B4-B3-B2), False Color (B8-B4-B3), NDVI, NDWI, MNDWI, NDBI in a 2×3 grid, plus a spectral profile chart (mean ± std per band).
- *SAR*: VV, VH, color composite (R=VV, G=VH, B=VV/VH), VV−VH difference map, plus a dual-density backscatter histogram.

**`compare_idx`** — Side-by-side spectral index comparison (NDVI, NDWI, MNDWI, NDBI) for optical data.

**`label`** — Four-panel label inspection: raw band, colored mask, morphological boundary overlay (dilation − erosion), and a pie chart showing class distribution with pixel counts.

**`metadata`** — Text-based metadata summary: file info, CRS, resolution, bounds, transform matrix, compression, and a per-band statistics table with min/max/mean/std/median/NaN%.

### Flexible Dataset Access

Five entry points, each suited to a different folder structure:

| Entry Point | Input | Best For |
|-------------|-------|----------|
| `inspect_tif(path, mode)` | Single file path | Quick one-off checks |
| `compare_tifs([p1, p2, ...])` | List of 2–4 paths | Cross-sensor or cross-ROI comparison |
| `FlatFolderBrowser(folder)` | Directory path | Folder with `.tif` files directly inside |
| `TiffBrowser(root_dir)` | Root directory path | `root/roi_name/*.tif` nested structure |
| `glob.glob()` + loop | Glob pattern | Scattered files across subdirectories |

---

## Requirements

```
numpy
rasterio
matplotlib
ipywidgets
scipy          # optional, for boundary detection in label mode
```

The notebook was developed and tested on Python 3.13 with the `hydroAI` conda environment.

---

## Quick Start

### 1. Open the notebook

```bash
jupyter notebook datacheck.ipynb
```

### 2. Run §0 (Visualization Engine)

This loads all shared functions and classes. Run it once per session.

### 3. Pick your entry point

**Single file — just see it:**

```python
inspect_tif("/path/to/Bolivia_S2.tif", mode="rgb")
```

**Browse a folder:**

```python
browser = FlatFolderBrowser("/data/my_region/optical/")
browser.show()
```

**Nested ROI structure with interactive widgets:**

```python
viewer = TiffBrowser("/data/Sen1Floods11/HandLabeled")
viewer.show()
```

**Batch process with glob:**

```python
import glob
for f in sorted(glob.glob("/data/**/*S2*.tif", recursive=True)):
    inspect_tif(f, mode="metadata")
```

**Compare files side by side:**

```python
compare_tifs([
    "/data/roi/optical.tif",
    "/data/roi/sar.tif",
    "/data/roi/label.tif",
], band_idx=0)
```

---

## Notebook Structure

```
datacheck.ipynb
│
├── §0  Visualization Engine        ← Run this first. All functions & classes.
│       - identify_data_type()         Auto-detection logic
│       - inspect_tif()                Main single-file function
│       - compare_tifs()               Multi-file comparison
│       - TiffBrowser                  Widget for nested ROI folders
│       - FlatFolderBrowser            Widget for flat folders
│
├── §1  Quick Look                  ← Single file, one function call
├── §2  Flat Folder Browser         ← Widget for a directory of .tif files
├── §3  Nested ROI Browser          ← Widget for root/roi_name/*.tif
├── §4  Glob Pattern Search         ← Batch inspection via pattern matching
└── §5  Multi-Path Comparison       ← 2-4 files side by side
```

---

## Supported Folder Structures

The toolkit does not impose any single directory convention. Examples of what works:

```
# Flat — files in one directory
optical/
├── 20240101.tif
├── 20240201.tif
└── 20240301.tif

# Nested ROI — one subfolder per region
HandLabeled/
├── Bolivia/
│   ├── Bolivia_103757_S2Hand.tif
│   ├── Bolivia_103757_S1Hand.tif
│   └── Bolivia_103757_LabelHand.tif
├── India/
│   └── ...
└── USA/
    └── ...

# Structured — subfolders by data type
Bolivia_23014/
├── optical/
│   └── Bolivia_23014_S2.tif
├── sar/
│   └── Bolivia_23014_S1.tif
└── label/
    └── Bolivia_23014_water.tif

# Scattered — use glob to find them
project/
├── region_a/
│   └── 2024/
│       └── S2_cloud_free.tif
└── region_b/
    └── 2023/
        └── S2_cloud_free.tif
```

---

## `inspect_tif()` Mode Reference

```python
inspect_tif(path, mode="auto")
```

| Mode | Description |
|------|-------------|
| `"auto"` | Picks the best mode based on detected data type |
| `"single"` | Band 1 image + histogram + CDF + stats |
| `"all"` | Grid of all bands + correlation heatmap |
| `"rgb"` | Optical composites or SAR composites (sensor-aware) |
| `"compare_idx"` | NDVI / NDWI / MNDWI / NDBI side by side |
| `"label"` | Colored mask + boundary + class distribution pie chart |
| `"metadata"` | CRS, resolution, per-band statistics table |

Auto mode defaults:
- Optical → `"rgb"`
- SAR → `"rgb"`
- Water Label / Cloud / Dynamic World → `"label"`
- Unknown → `"all"`

---

## Spectral Indices

All indices are computed from Sentinel-2 TOA bands:

| Index | Formula | Bands | Purpose |
|-------|---------|-------|---------|
| NDVI | (NIR − Red) / (NIR + Red) | B8, B4 | Vegetation |
| NDWI | (Green − NIR) / (Green + NIR) | B3, B8 | Open water |
| MNDWI | (Green − SWIR1) / (Green + SWIR1) | B3, B11 | Water (urban-robust) |
| NDBI | (SWIR1 − NIR) / (SWIR1 + NIR) | B11, B8 | Built-up area |

---

## Author

**Beomsik Kim**
Dept. of Geoinformatics, University of Seoul / GIST Hydro AI Lab Intern
2026
