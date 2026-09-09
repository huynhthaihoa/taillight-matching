# Taillight Matching and Pairing Algorithm

Source code for the paper ["A Taillight Matching and Pairing Algorithm for Stereo-Vision-Based Nighttime Vehicle-to-Vehicle Positioning"](https://www.mdpi.com/2076-3417/10/19/6800/htm).

## Overview

This project implements an algorithm to detect, match, and pair **vehicle taillight LEDs** in stereo nighttime images for relative positioning between vehicles. The system processes stereo image pairs, identifies red LED regions, matches taillights across left and right images (stereo matching), and pairs multiple taillights within individual images to determine vehicle geometry.

### Key Features
- **LED Detection**: HSV color thresholding followed by connected-component labeling to isolate red taillight regions
- **Stereo Matching**: Matches corresponding taillight regions between left and right camera views using NCC, histogram comparison, and feature descriptors (SIFT/SURF/ORB)
- **Taillight Pairing**: Identifies pairs of taillights within a single image (left and right lights of one vehicle)
- **Feature Extraction**: Computes geometric and photometric features (position, size, centroid distance) for both matching and pairing tasks
- **Dataset Generation**: Automatically creates labeled Excel datasets for machine learning model training

## Repository Structure

### Core Modules

| File | Purpose |
|------|---------|
| **Region.py** | Defines `CRegion` (single LED region) and `CList` (collection of regions) classes. Handles region properties (boundaries, centroids), feature descriptors (HOG, SIFT, SURF, ORB), and similarity metrics (NCC, Bhattacharyya distance). |
| **simulation.py** | Main entry point. Orchestrates the complete pipeline: color thresholding → label assignment → feature extraction → stereo matching/pairing. Generates dataset Excel files. |
| **tool.py** | Data processing utilities for Excel dataset manipulation: normalization, filtering, ground truth labeling, removing bus vehicle records. |
| **utils.py** | System-wide parameters and constants: HSV thresholds, feature descriptors, ground truth mappings (stereo matches and taillight pairs per image). |

## How It Works

### 1. LED Detection
- **Color Thresholding**: Converts BGR images to HSV and applies range masking for red taillights
  - Hue range 1: 0° to 9° (red in HSV)
  - Hue range 2: 342° to 360° (red wrap-around)
  - Configurable saturation and value thresholds in `utils.py`

- **Connected-Component Labeling**: Assigns labels to foreground pixels using a two-pass algorithm (see `assignLabel()` in `simulation.py`)

- **Region Suppression**: Filters out noise and brake lamps based on:
  - Minimum region size (default: 10 pixels)
  - Height position (removes regions too high in image)
  - Aspect ratio (removes extremely wide or tall regions)

### 2. Feature Extraction (per Region)
Each detected LED region is characterized by:
- **Geometric Features**:
  - Distance to border (DBL/DBR for stereo, normalized by image dimensions)
  - Width, height, area (normalized by image size)
  - Vertical position
  - Distance to nearest neighbor region

- **Photometric Features**:
  - NCC (Normalized Cross-Correlation Histogram): 16-bin HSV histogram comparison
  - Bhattacharyya distance: Statistical divergence between histograms
  - Optional descriptors: Dense SIFT, SURF, ORB keypoints

### 3. Stereo Matching
Matches each left-image region to a corresponding right-image region using:
- NCC histogram similarity
- Geometric constraints (position, size ratios)
- Optional feature descriptor matching

Output: Pairs of (left_region_id, right_region_id) confirmed as the same taillight viewed from two cameras.

### 4. Taillight Pairing (Within Single Image)
Groups two regions in the same image (left and right taillights of one vehicle) by:
- Inter-distance between regions (horizontal spacing)
- NCC similarity (should look similar)
- Vertical centroid ratio (should be roughly aligned vertically)

Output: Pairs of (region_id_1, region_id_2) representing the two taillights of one vehicle.

## Usage

### Prerequisites
```bash
pip install opencv-python opencv-contrib-python numpy openpyxl scikit-image scikit-learn scipy matplotlib
```

### Running the Pipeline

1. **Organize input images**:
   ```
   Dataset/Set_2/
   ├── left/
   │   ├── 1.JPG
   │   ├── 2.JPG
   │   └── ...
   └── right/
       ├── 1.JPG
       ├── 2.JPG
       └── ...
   ```

2. **Run the main script**:
   ```bash
   python simulation.py
   ```
   
   This generates three Excel files:
   - `Set_2/matching_Set_2.xlsx` — stereo matching dataset (14 features per row)
   - `Set_2/pairing_left_Set_2.xlsx` — pairing dataset for left images (18 features per row)
   - `Set_2/pairing_right_Set_2.xlsx` — pairing dataset for right images (18 features per row)

3. **(Optional) Data Cleaning & Normalization**:
   Edit `tool.py` to call utility functions:
   ```python
   normalizeDistance('Set_2/matching_Set_2.xlsx', sheet_id=0, col=14, getAbs=True)
   verticalDistance('Set_2/matching_Set_2.xlsx', sheet_id=0, col=15, getAbs=True)
   generateStereoDataset('Set_2/matching_Set_2.xlsx', sheet_id=0, col=13, 
                         stereo_matching_left, stereo_matching_right)
   ```

### Key Functions

**Detection & Feature Extraction** (simulation.py):
- `detectLEDRegion(filename)` — Full detection pipeline for a single image
- `thresholdColor(img)` — Apply HSV color mask
- `assignLabel(img, origin_img)` — Connected-component labeling
- `generateMatchingDataset()` — Create stereo matching training data
- `generatePairingDataset()` — Create taillight pairing training data

**Data Processing** (tool.py):
- `normalizeDistance()` — Normalize inter-distances by image width
- `generateStereoDataset()` — Label dataset rows based on ground truth
- `removeBus()` — Filter out bus vehicle records from dataset
- `parseGroundtruthData()` — Load ground truth from INI files

### Customization

Modify `utils.py` to adjust:
```python
# Color thresholds (HSV)
min_H_1, max_H_1      # Hue range 1 (red low end)
min_H_2, max_H_2      # Hue range 2 (red high end)
min_S, max_S          # Saturation range
min_V, max_V          # Value range

# Region suppression thresholds
SIZE_THRES = 10       # Minimum region size (pixels)
HEIGHT_THRES = 3/5    # Minimum height position (fraction of image height)
BRAKE_THRES = 5       # Maximum aspect ratio (filters tall outliers)

# Feature descriptor type
class FD(Enum):
    DENSE_SIFT, DENSE_SURF, DENSE_ORB, etc.
```

## Output Formats

### Excel Dataset Structure

**Stereo Matching** (matching_Set_2.xlsx):
```
| Image | Left_ID | Right_ID | DB_L | W_L | H_L | V_L | DN_L | DB_R | W_R | H_R | V_R | DN_R | NCC | ... |
```

**Taillight Pairing** (pairing_left_Set_2.xlsx / pairing_right_Set_2.xlsx):
```
| Image | ID_1 | ID_2 | W_1 | H_1 | V_1 | W_2 | H_2 | V_2 | Inter_Distance | NCC | NCC_2 | Bhatta | ... |
```

## Paper Reference

**Title**: Taillight Matching and Pairing Algorithm for Stereo-Vision-based Nighttime Vehicle-to-Vehicle Positioning  
**Journal**: Applied Sciences, 2020  
**DOI**: [10.3390/app10196800](https://www.mdpi.com/2076-3417/10/19/6800/htm)

## Notes

- Ground truth data (stereo matches and taillight pairs) are hardcoded in `utils.py` as dictionaries (`stereo_matching_left`, `stereo_matching_right`, `taillight_pairing_left`, `taillight_pairing_right`)
- Bus vehicles are filtered separately via `bus_map_left` and `bus_map_right` in `utils.py`
- The code is currently Windows-path dependent (backslashes); cross-platform support can be added by replacing path strings with `os.path.join()`
- Feature descriptors (**SIFT**/**SURF**/**ORB** computation) are optional and disabled by default in dataset generation (set to 0)
