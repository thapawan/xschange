# xschange

**Automated Extraction and Visualization of Terrain Cross‑Sections from Line Features and Digital Elevation Models**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.9+](https://img.shields.io/badge/python-3.9+-blue.svg)](https://www.python.org/downloads/)

**xschange** is a Python‑based tool that extracts elevation cross‑sections from vector line features and raster DEMs in batch mode. It exports both tabular (CSV) and graphical (PNG) outputs, enabling reproducible and scriptable terrain profiling workflows.

## Features

- Batch processing of any number of cross‑section lines
- User‑controlled sampling interval (number of points per transect)
- Automatic horizontal distance calculation along each transect
- Bilinear interpolation for smooth elevation sampling
- Exports to CSV and PNG for each cross‑section
- ArcGIS Pro Python Toolbox interface + standalone Jupyter notebooks
- Handles projected coordinate systems only (explicit error for geographic coordinates)

## Installation

### Using conda (recommended)

```bash
conda env create -f environment.yml
conda activate csev

