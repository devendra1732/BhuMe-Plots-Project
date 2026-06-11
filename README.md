# BhuMe — Cadastral Plot Correction

**Take-home submission for BhuMe land records engineering role**

---

## The Problem

Maharashtra's cadastral maps were drawn over a century ago using chain and plane-table surveys. When digitised and georeferenced onto modern satellite imagery by MRSAC, the fitting process introduced drift — plot boundaries are often displaced from the real fields they describe by 10–50 metres. This project automatically detects and corrects that drift.

---

## Approach

The pipeline runs in three passes on any village bundle (`input.geojson` + `imagery.tif` + `boundaries.tif`):

### Pass 1 — Classification by area ratio
For each plot, compute `drawn_area / recorded_area`:
- **0.70 – 1.40** → placement candidate (shape is right, position is wrong)
- **0.40 – 0.70 or 1.40 – 2.10** → borderline (attempt with penalty)
- **< 0.40 or > 2.10** → area problem (flag immediately — moving won't fix it)

### Pass 2 — Per-plot offset via cross-correlation
For each placement candidate:
1. Extract a pixel patch from `boundaries.tif` under the plot bounding box (template)
2. Extract an expanded search window (40% padding) from `imagery.tif`
3. Slide the template over the search window using normalised cross-correlation (NCC)
4. Convert the peak pixel shift → metres → lon/lat delta using Web Mercator (EPSG:3857)

### Pass 3 — Confidence scoring + village median fallback
- **Confidence** = weighted combination of NCC peak sharpness (peak/second-peak ratio) and area ratio quality
- Plots below confidence threshold (0.35) are **flagged**, not corrected
- Village-level **median offset** computed from all confident corrections, used as fallback for weak-signal plots

---

## Results

| Village | Plots | Corrected | Flagged | Skipped | Median drift |
|---|---|---|---|---|---|
| Vadnerbhairav | 2,457 | 1,809 | 264 | 80 | ~34 m SW |
| Malatavadi | 2,508 | 0 | 2,172 | 55 | boundaries.tif mismatch |

> **Note on Malatavadi**: The `boundaries.tif` file covers Vadnerbhairav's coordinates, not Malatavadi's. Without a valid edge-hint layer, cross-correlation signals were too weak village-wide. The pipeline correctly flagged rather than guessing.

---

## Confidence Design

Confidence is deliberately variable — not flat. A flat score produces 0.5 AUC (random). Each plot's confidence reflects:

- **NCC peak sharpness**: a clean unique peak = high confidence; a flat/noisy response = low
- **Area ratio proximity to 1.0**: closer to 1.0 = more likely a pure translation error
- **Classification penalty**: borderline plots are penalised before thresholding

---

## Running the Code

```bash
# Install dependencies
pip install numpy scipy

# Run on a village directory
python correct_plots_v2.py /path/to/village/

# Or from the same directory as the data files
python correct_plots_v2.py
```

**Input** (in the village directory):
```
input.geojson
imagery.tif
boundaries.tif
```

**Output** (written beside the inputs):
```
predictions.geojson
```

No geospatial dependencies required (`rasterio`, `geopandas`, `fiona` are NOT needed). The script reads GeoTIFF files directly using Python's `struct` module and handles Web Mercator coordinate conversion natively.

---

## Output Format

Each feature in `predictions.geojson`:

```json
{
  "type": "Feature",
  "geometry": { ... },
  "properties": {
    "plot_number": "142",
    "status": "corrected",
    "confidence": 0.7812,
    "method_note": "xcorr shift (4px,-6px)=(9.6m,-14.3m) peak=0.821 ratio=1.031 conf=0.781"
  }
}
```

| Field | Values |
|---|---|
| `status` | `corrected` or `flagged` |
| `confidence` | 0–1, only present on corrected plots |
| `method_note` | pixel shift, metre offset, NCC peak, area ratio |

---

## Repository Structure

```
/
├── correct_plots_v2.py          # Main pipeline (runs both villages)
├── vadnerbhairav/
│   └── predictions.geojson      # Village 1 output
├── malatavadi/
│   └── predictions.geojson      # Village 2 output
├── transcripts/
│   └── README.md                # Links to AI conversation transcripts
└── README.md
```

---

## What Would Improve It (Gold / Platinum)

- **Calibration curve**: fit confidence → IoU using known ground-truth plots so scores are probabilistically meaningful
- **scipy fftconvolve**: faster, higher-resolution cross-correlation for small plots
- **Neighbour consistency**: adjacent plots should have similar offsets; outliers can be revised
- **Automatic raster validation**: detect CRS/coverage mismatches before processing
- **Multi-scale search**: coarse-to-fine offset estimation to handle large drifts

---

## Author

Submitted for BhuMe engineering take-home · [yash@bhume.in](mailto:yash@bhume.in)
