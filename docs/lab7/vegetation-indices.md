# Interpreting vegetation indices

Vegetation indices are mathematical combinations of two or more spectral bands that compress different information about vegetation into a single number. A widely used vegetation index is the normalised difference vegetation index (NDVI) which captures information about canopy chemical composition (red light absorption by chlorophyll) and structure (NIR scattering by spongy mesophyll cells). 

$$NDVI = \frac{NIR - red}{NIR + red}$$

However, a limit of the NDVI is that it saturates at high biomass levels. Once green vegetation cover reaches a certain density adding more leaf layers to the canopy doesn't register as an increase in NDVI values. This limits the use of the NDVI over certain ecosystems (e.g. dense forests) and agricultural applications (e.g. cpaturing variability in productive crops at peak biomass). 

This saturation of NDVI values at high biomass is due to two reasons:

1) Compressing canopy information into finite bounds between [-1, 1]. Over dense canopies, shifts in NDVI resulting from increasing biomass need to be squeezed in below a limit of 1 resulting in very small increases in NDVI values.

2) Chlorophyll in the upper leaf layers absorb all the red light. Adding more leaf canopy below doesn't register a change in red reflectance. Therefore, red falls out of the NDVI formula which tends towards a value of 1 ($NIR / NIR$ = 1). This is problematic as NIR reflecance will keep increasing with canopy when the red signal drops off and the NDVI formula doesn't capture this information. 

### EVI

The enhanced vegetation index (EVI) has been developed to capture the signal of increases in NIR reflectance over dense canopies when chlorophyll has absorbed all the red light. 

$$EVI = G \times \frac{NIR - red}{{NIR} + C_1{red} - C_2{blue} + L}$$

$$EVI = 2.5 \times \frac{{NIR} - {red}}{{NIR} + 6{red} - 7.5{blue} + 1}$$

The L in the denominator is a constant (typically 1 for EVI). When red reflectance tends towards zero, the denominator remains $NIR + 1$ so increases in NIR still register. As NIR reflectance corresponds to scattering by cells within leaves and leaves within the canopy, it's more sensitive to the structure of the canopy. 

### NDRE

The normalised difference red edge index swaps the red band for the red edge band. The red edge band is a slightly longer wavelength than red light (705 nm on Sentinel-2 for band 5) and corresponds to the portion of vegetation's spectral profile where reflectance ramps up in between red and NIR wavelengths. 

Compared to red light, "red edge" light can penetrate further into canopies before it is fully absorbed. Therefore, it can capture more information about leaf and canopy chlorophyll content after the upper leaf layers have absorbed all the red light. One way of thinking of the NDRE is that it's similar to the NDVI but with a larger dynamic range and able to capture more variability at high cholorphyll levels. The NDRE cannot tell the difference between more chlorophyll due to higher concentrations or more leaves. 

$$NDRE =  \frac{NIR - red_{edge}}{NIR + red_{edge}}$$

### Setup

This lab will compare Sentinel-2 NDVI, EVI and NDRE over a forest region in the North West of the USA and demonstrate the NDVI saturation effect. The focus of this lab is:

* interpreting remote sensing images and vegetation indices. 
* practicing data visualisation in Google Earth Engine using charts and the map display. 

Create a new script in your *labs-gee/lab-7* repository called *vegetation-indices.js*. Enter the following comment header to the script. 

```js
/*
Vegetation indices
Author: Test
Date: XX-XX-XXXX

*/
```

Load the area of interest for this lab:

```js
var aoi = ee.Geometry.Polygon(
        [[[-123.55550806624096, 47.04660205755613],
          [-123.55550806624096, 46.988787269731745],
          [-123.33440821272534, 46.988787269731745],
          [-123.33440821272534, 47.04660205755613]]]);
var region = ee.Geometry.Polygon(
        [[[-123.8016704807914, 47.08394273869796],
          [-123.8016704807914, 46.87171926146504],
          [-123.0106548557914, 46.87171926146504],
          [-123.0106548557914, 47.08394273869796]]]);
Map.centerObject(aoi, 14);
```

`aoi` refers to a targeted area for for assessment of different vegetation indices. `region` is broader spatial extent to see landscape context. 

First, set some values that you will need to reuse throughout the script. 

```js
var startDate = '2023-01-01';
var endDate = '2024-01-01';
var eviParams = {G: 2.5, C1: 6, C2: 7.5, L: 1};
```

## Sentinel-2 data

Load the Sentinel-2 surface reflectance `ImageCollection` and drop all scenes with greater than 40 % cloud cover.

```js
var s2 = ee.ImageCollection('COPERNICUS/S2_SR_HARMONIZED')
    .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 40));
```

<details>
  <summary><b>Can you generate code to filter Sentinel-2 <code>ImageCollection</code> to the date range specified by <code>startDate</code> and <code>endDate</code> and the extent of the <code>region</code> geometry?</b></summary>
  <p>
```js
s2 = s2.filterBounds(region)
    .filterDate(startDate, endDate);
```
  </p>
</details>

The following function can be mapped over the `s2` `ImageCollection` to mask cloudy pixels.

```js
// Function to mask clouds using the Sentinel-2 QA band
function maskS2clouds(img) {
  var qa = img.select('QA60');

  // Bits 10 and 11 are clouds and cirrus, respectively.
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;

  // Both flags should be set to zero, indicating clear conditions.
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
      .and(qa.bitwiseAnd(cirrusBitMask).eq(0));

  return img.updateMask(mask)
    .divide(10000)
    .copyProperties(img, ['system:time_start']);
}
```

<details>
  <summary><b>Can you write functions to add NDVI, EVI and NDRE bands to the <code>ImageCollection</code> and to clip each image to the extent of <code>region</code>? Map all three functions (cloud masking, vegetation index computation and image clipping) over the <code>s2</code> <code>ImageCollection</code>. Hint: <code>clamp</code> the EVI values between [-1, 1.2] -  look up how to do this in the Google Earth Engine docs or use an LLM to help.</b></summary>
  <p>

```js
function clipImg(img) {
  return img.clip(region);
}

function addIndices(img) {
  var nir  = img.select('B8');
  var red  = img.select('B4');
  var blue = img.select('B2');
  var re   = img.select('B5'); // red-edge 1 (~705 nm)

  var ndvi = nir.subtract(red).divide(nir.add(red)).rename('NDVI');
  var evi  = nir.subtract(red)
                .divide(nir.add(red.multiply(eviParams.C1))
                           .subtract(blue.multiply(eviParams.C2)).add(eviParams.L))
                .multiply(eviParams.G).clamp(-1, 1.2).rename('EVI');
  var ndre = nir.subtract(re).divide(nir.add(re)).rename('NDRE');

  return img.addBands([ndvi, evi, ndre]);
}

s2 = s2.map(maskS2clouds).map(addIndices).map(clipImg);
```
</p>
</details>

## Peak VI images

`s2` stores a year's worth of Sentinel-2 images that have been cloud masked and had NDVI, EVI and NDRE bands added. To explore the NDVI saturation effect create peak vegetation index value images and visualise them on the map display. 

The following code snippet will create an `Image` with bands for the maximum pixel-wise NDVI, EVI and NDRE value over the year. There are some noisy retrievals over dark objects where the vegetation index values equal 1. These are masked out. 

```js
var peakImg = s2.select(['NDVI', 'EVI', 'NDRE']).qualityMosaic('NDVI');
peakImg = peakImg.updateMask(peakImg.select(['NDVI']).neq(1));
peakImg = peakImg.updateMask(peakImg.select(['EVI']).neq(1));
peakImg = peakImg.updateMask(peakImg.select(['NDRE']).neq(1));
```

**Look up `qualityMosaic()` in the Google Earth Engine docs - how does this function generate a peak NDVI `Image`?**

<details>
  <summary><b>Can you add each of the peak vegetation index <code>Image</code> to the map? What are your initial impressions of the variability in vegetation cover and canopy condition captured by each of the three indices? In particular, focus on areas of dense forest. Hint: turn the basemap to satellite images to use as a reference.</b></summary>
  <p>

```js
var viVis   = {min: 0, max: 1, palette: ['ffffff','c2e699','78c679','238443','004529']};
var eviVis  = {min: 0, max: 1, palette: ['ffffff','fdd49e','fc8d59','d7301f']};
var ndreVis = {min: 0, max: 1, palette: ['ffffff','dadaeb','9e9ac8','6a51a3']};

// Peak-canopy maps
Map.addLayer(peakImg.select('NDVI'), viVis,   'PEAK NDVI');
Map.addLayer(peakImg.select('EVI'),  eviVis,  'PEAK EVI',  false);
Map.addLayer(peakImg.select('NDRE'), ndreVis, 'PEAK NDRE', false);
```
</p>
</details>

## Data visualisation

While it is possible to get a general impression of the relationship between the three vegetation indices and whether NDVI is saturating over certain land covers or vegetation contexts, these effects are easier to spot using scatter plots comparing two indices. 

Let's generate a sample of points across the focused assessment area `aoi` to compare relationships between the vegetation indices. The below code snippet will randomly sample 1500 points within the bounds of `aoi` and extract the NDVI, EVI and NDRE values. 

```js
var peakSample = peakImg.select(['NDVI', 'EVI', 'NDRE',]).sample({
  region: aoi, scale: 20, numPixels: 1500,
  geometries: false, dropNulls: true
});
print('Inspect the peak sample:', peakSample.first());
```

This is a convenience function that plots three scatter plots. 

```js
// Helper: print the three scatters for one filtered subset
function threeScatters(fc, label) {
  print(ui.Chart.feature.byFeature(fc, 'NDVI', ['EVI'])
    .setChartType('ScatterChart').setOptions({
      title: label + ' \u2014 EVI vs NDVI',
      hAxis: {title: 'NDVI'}, vAxis: {title: 'EVI'},
      pointSize: 2, trendlines: {0: {showR2: true, visibleInLegend: true}}}));

  print(ui.Chart.feature.byFeature(fc, 'NDVI', ['NDRE'])
    .setChartType('ScatterChart').setOptions({
      title: label + ' \u2014 NDRE vs NDVI',
      hAxis: {title: 'NDVI'}, vAxis: {title: 'NDRE'},
      pointSize: 2, trendlines: {0: {showR2: true, visibleInLegend: true}}}));

  print(ui.Chart.feature.byFeature(fc, 'NDRE', ['EVI'])
    .setChartType('ScatterChart').setOptions({
      title: label + ' \u2014 EVI vs NDRE',
      hAxis: {title: 'NDRE'}, vAxis: {title: 'EVI'},
      pointSize: 2, trendlines: {0: {showR2: true, visibleInLegend: true}}}));
}
```

<details>
  <summary><b>Can you pass <code>peakSample</code> into it and generate scatter plots showing: NDVI v EVI, NDVI v NDRE and NDRE v EVI?</b></summary>
  <p>
```js
threeScatters(peakSample, 'Peak VI');
```
</p>
</details>

Inspect the scatter plots that you have generated and answer the following questions:

<details>
  <summary><b>What does each point on the chart represent and how do you interpret the shape of the cloud of points?</b></summary>
  <p>
Each point is a pixel represented in space defined by its value in two vegetation indices. The shape of the cloud tells you about the relationship between two vegetation indices. If they are linear both indices capture the same information. If they start to bunch up or you get a vertical smearing then it implies one index has saturted and the other one contains new information. 
</p>
</details>

<details>
  <summary><b>On the NDVI-EVI plots describe the pattern once NDVI values reach 0.8?</b></summary>
  <p>
The linear relationship between NDVI and EVI breaks down and NDVI values bunch and spread vertically. This is NDVI saturating and losing the ability to capture variability between high biomass locations. However, the EVI remains sensitive to changes in vegetation condition (structure). 
</p>
</details>

<details>
  <summary><b>Why does the EVI keep rising once the NDVI values saturate?</b></summary>
  <p>
The EVI is weighted towards the NIR signal which responds to scattering off leaves and leaf layers. As more leaf layers are added there is more scattering and NIR reflectance increases. In other words, EVI is sensitive to canopy structure. However, at these dense canopy levels the upper layers of the canopy have absorbed all the red light and so the NDVI values collapse towards 1.  
</p>
</details>

<details>
  <summary><b>Which stays more linear at high-NDVI values - the NDVI-EVI plot or the NDVI-NDRE plot?</b></summary>
  <p>
The NDVI-NDRE plot. However, there is still evidence of the NDVI bunching up towards one but to a lesser extent than on the NDVI-EVI plot. 
</p>
</details>

<details>
  <summary><b>Which stays more linear the NDVI-EVI or NDRE-EVI plot at high biomass levels?</b></summary>
  <p>
The NDRE-EVI plot. This is because the red-edge band is more sensitive to chlorophyll content in dense, green canopies. However, the bunching effect is still apparent. The NDRE still saturates in dense canopies it can just go a bit further than NDVI. It seems to saturate at a value of 0.8 here. There is a loose linear fit between the NDRE and EVI. This implies they are both tracking greenness but capturing different aspects of it (EVI - structure) and (NDRE - chlorophyll).
</p>
</details>

<details>
  <summary><b>Which of these indices is most useful to a forest manager?</b></summary>
  <p>
Not NDVI here - this index saturates. The EVI is useful to provide information on canopy structure and how much canopy. The NDRE is useful capturing information about chlorophyll content and early warning for stress and health.  
</p>
</details>

## Land cover and VI analysis

This scene is a mix of farmland and forest. It is possible that some of the scatter and patterns on these plots is an artefact of the mix of land covers. Looking at the NDRE-EVI plot, and holding an NDRE value fixed, observing a range of EVI values could be interpreted as a fixed level of greenness (chlorophyll content) across a range of canopy densities. However, this interpretation holds within a single land cover type (e.g. just within forest canopies). It might be confounded by mixing multiple land covers on a plot where each land cover has a different NDRE-EVI relationship. It is important to make sure the apparent decoupling and spread between NDRE-EVI values is a real feature of the canopy and not an artefact of mixing two land covers' NDRE-EVI relationships on the same plot. To assess this, these plots should be generated for different land cover types. 

The following code snippet loads a 10 m spatial resolution land cover map and creates a forest and a cropland / grassland mask.

```js
var worldcover = ee.ImageCollection('ESA/WorldCover/v200').first()
                   .clip(region);

// Remap to compact codes: 1 = forest, 2 = cropland, 3 = grassland.
// remap() masks everything not listed, so only these three classes remain.
var lc = worldcover.remap([10, 40, 30], [1, 2, 3]).rename('lc');

// Attach the class to the peak-VI image
var peakWithLC = peakImg.addBands(lc);

Map.addLayer(lc.selfMask(),
  {min: 1, max: 3, palette: ['238443', 'e6550d', 'c2e699']},
  'Land cover (1=forest, 2=crop, 3=grass)', false);
```

**Homework activities**

Can you use this land cover map to filter out only forest pixels or only grassland / cropland pixels and regenerate the NDVI-EVI, NDVI-NDRE and NDRE-EVI scatter plots for each land cover group? Hint: you will need to sample points from the `peakWithLc` `Image`. 

How do these scatter plots compare to the pooled scatter plots?

The in-built cloud mask with Sentinel-2, and used here, has known limitations. Can you edit this workflow to use the <a href="https://developers.google.com/earth-engine/datasets/catalog/COPERNICUS_S2_CLOUD_PROBABILITY" target="_blank">Sentinel-2: Cloud Probability</a> product? How do your results change? This is a good opportunity to practice using LLMs to develop Google Earth Engine workflows and verify the outputs. 