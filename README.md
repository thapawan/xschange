# xschange

**xschange** is an open‑source Python library for **temporal alignment and quantitative change attribution of DEM‑derived cross‑sections**.  
It enables reproducible, batch analysis of how terrain cross‑sections evolve over time, addressing a long‑standing gap between GIS‑based visualization tools and research‑oriented analytical workflows.

---

## Motivation

Digital Elevation Models (DEMs) are increasingly available at high temporal frequency, yet most GIS software supports cross‑section analysis only as **single‑time visualization**. Comparing cross‑sections across time is typically performed manually or using ad‑hoc scripts, making results difficult to reproduce and hard to scale.

xschange was developed to support **analysis‑first workflows**, where cross‑sectional change can be:
- quantified consistently,
- processed in batch,
- and integrated directly into statistical or machine‑learning pipelines.

---

## Key Features

✅ Temporal normalization of cross‑sections  
✅ Standardized change attribution metrics (cut/fill, asymmetry, net change)  
✅ Batch processing across many cross‑sections and time steps  
✅ Analysis‑ready tabular outputs (CSV / Parquet)  
✅ Headless execution (no GUI dependency)  
✅ Designed for reproducible research

---

## What xschange Is (and Is Not)

### xschange **is**
- A research‑oriented engine for **quantifying cross‑sectional change through time**
- Designed for geomorphology, hydrology, and environmental analysis

### xschange **is not**
- A replacement for ArcGIS or QGIS visualization tools
- A GUI‑driven profiling plugin
- A hydraulic or sediment transport model

xschange complements existing GIS tools by focusing on **quantitative change attribution**, not interactive visualization.

---

## Installation

### Using Conda (recommended)

```bash
conda env create -f environment.yml
conda activate xschange
pip install -e .

from xschange import process_cross_section_change

results = process_cross_section_change(
    cross_sections="data/cross_sections.shp",
    dem_stack={
        2015: "data/dem_2015.tif",
        2024: "data/dem_2024.tif"
    },
    spacing=5,
    reference="centerline"
)
---

## Tests

Example:

```python
def test_normalized_length_consistency():
    """
    Normalized cross-sections should have identical sample counts
    regardless of input resolution.
    """
    ...
