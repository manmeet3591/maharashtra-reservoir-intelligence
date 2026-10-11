# Maharashtra Reservoir Intelligence

**Satellite-derived water & drought monitoring dashboard** for Maharashtra, India.  
Produced by the [DSOC AI Research Lab](mailto:dsocai.students@wku.edu), Western Kentucky University.

🔗 **Live dashboard:** https://manmeet3591.github.io/maharashtra-reservoir-intelligence/

---

## What this shows

The Maharashtra state government declared drought in 2026. This dashboard makes that visible from space using four satellite-derived indicators:

| Layer | Data source | Method |
|---|---|---|
| **Drought severity** | Composite index | Per-district classification |
| **Reservoir levels** | CWC + Sentinel-2 | Surface water extent → storage % |
| **NDVI anomaly** | MODIS MOD13A2 | 2026 value − 2001–2020 climatological mean |
| **Dynamic World land cover** | Google Dynamic World v1 | **Pixel-count first-order approximation** in GEE |

### First-order approximation explained

> Jasveer asked: "what can we actually do with satellite data for drought?"  

The simplest answer: **count pixels**.

In Google Earth Engine, Dynamic World assigns every 10 m pixel a land-cover label (water, crops, bare ground, vegetation, …). Summing pixels by class across the state boundary gives us fractional area — no field surveys, no model assumptions. This "first-order approximation" is:

- Fast to compute (seconds in GEE)
- Fully reproducible
- Easy to audit and explain to government stakeholders

More advanced: satellite-derived **bathymetry** uses NIR and SWIR reflectance band ratios over reservoir footprints to estimate relative water depth, calibrated against a full-pool reference image.

---

## GEE Code Snippet

The core Dynamic World pixel-counting logic (paste into [code.earthengine.google.com](https://code.earthengine.google.com)):

```javascript
// Maharashtra boundary
var maharashtra = ee.FeatureCollection('FAO/GAUL/2015/level1')
  .filter(ee.Filter.eq('ADM1_NAME', 'Maharashtra'));

// Dynamic World — latest composite
var dw = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1')
  .filterBounds(maharashtra)
  .filterDate('2026-09-01', '2026-10-15')
  .select('label')
  .mode()  // majority label per pixel
  .clip(maharashtra);

// DW class names (0–8)
var DW_CLASSES = ['water','trees','grass','flooded_veg','crops',
                  'shrub_scrub','built','bare','snow_ice'];

// Pixel-count per class (first-order approximation)
var areaImage = ee.Image.pixelArea().addBands(dw);
var areas = areaImage.reduceRegion({
  reducer: ee.Reducer.sum().group({groupField: 1, groupName: 'class'}),
  geometry: maharashtra,
  scale: 100,
  maxPixels: 1e10
});

print('Land cover areas (m²):', areas);

// Water bodies change detection (2025 vs 2026)
function getWaterFraction(startDate, endDate) {
  return ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1')
    .filterBounds(maharashtra)
    .filterDate(startDate, endDate)
    .select('label')
    .mode()
    .eq(0)  // class 0 = water
    .clip(maharashtra);
}

var water2025 = getWaterFraction('2025-09-01', '2025-10-15');
var water2026 = getWaterFraction('2026-09-01', '2026-10-15');
var waterLoss  = water2025.subtract(water2026);  // pixels that were water, now aren't

Map.centerObject(maharashtra, 7);
Map.addLayer(water2025, {palette:['white','#2060AA']}, 'Water 2025');
Map.addLayer(water2026, {palette:['white','#0020AA']}, 'Water 2026');
Map.addLayer(waterLoss,  {palette:['white','#C2321A']}, 'Water Loss 2025→2026');
```

### Satellite Bathymetry (Stumpf ratio method)

```javascript
// Estimate relative water depth from Sentinel-2 band ratio
var s2 = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')
  .filterBounds(maharashtra)
  .filterDate('2026-09-01', '2026-10-15')
  .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 10))
  .median();

// Stumpf (2003) log-ratio transform: ln(Green) / ln(Blue)
var green = s2.select('B3').divide(10000);
var blue  = s2.select('B2').divide(10000);
var bathyRatio = green.log().divide(blue.log()).rename('bathy_index');

// Higher values → shallower water
// Calibrate against known full-pool shoreline for absolute depth
Map.addLayer(bathyRatio.clip(maharashtra), {min:0.8,max:1.4,palette:['#084594','#2171B5','#6BAED6','#C6DBEF']}, 'Bathymetry index');
```

---

## Data sources

| Source | What it provides |
|---|---|
| [Google Dynamic World v1](https://developers.google.com/earth-engine/datasets/catalog/GOOGLE_DYNAMICWORLD_V1) | 10 m land cover, near-real-time |
| [MODIS MOD13A2](https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MOD13A2) | 16-day NDVI composites, 1 km |
| [IMD](https://mausam.imd.gov.in/) | District rainfall data |
| [CWC](https://cwc.gov.in/reservoir-storage) | Central Water Commission reservoir levels |
| [Copernicus S2](https://developers.google.com/earth-engine/datasets/catalog/COPERNICUS_S2_SR_HARMONIZED) | Sentinel-2 multispectral for bathymetry |

---

## Repo structure

```
maharashtra-reservoir-intelligence/
├── index.html     ← The full dashboard (self-contained, no build step)
└── README.md      ← This file
```

The dashboard is a **single static HTML file** — no Node, no build tools, no backend. Works on any static host (GitHub Pages, Netlify, Vercel, etc.).

---

*DSOC AI Research Lab · Western Kentucky University · dsocai.students@wku.edu*