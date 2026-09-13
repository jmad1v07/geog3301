# Extreme heat over India's croplands - trend analysis

India's North-West Indo-Gangetic Plains are a key food producing region typified by a rice (kharif / wet season) and wheat (rabi / dry season) cropping system. At the same time, these cropping systems are threatened by a range of climatic stressors.  In a <a href="https://www.nature.com/articles/nclimate1356" target="_blank">2012 *Nature Climate Change* study Lobell et al.,</a> showed that increased exposure to extreme heat during the grain filling period of wheat crop growth accelerated crop senescence and harms crop yields. A recent analysis by <a href="https://iopscience.iop.org/article/10.1088/1748-9326/acf871/pdf" target="_blank">Singh Sidhu (2023)</a> documented the effect of the 2022 heat waves during the grain filling phase on North-West India's wheat crop. 

This lab builds on Lobell et al., 2012's work and combines time-series of remote sensing data with daily global climate data to map trends in India's wheat crop extreme heat exposure during key growth stages. For all cropland pixels, time-series of vegetation indices (VI) will be used to identify the peak of the wheat crop's growing season and proxy the grain filling stage as the 45 days post-peak VI for each year. Then, an indicator of extreme heat exposure during each pixel-year combination's grain filling period will be computed using daily global climate data. Finally, annual trends in the exposure of wheat crops to extreme heat at this key growth stage will mapped over a 20 year period.

In this lab you will:

* extract phenology parameters from time-series of vegetation indices.
* map phenology parameters for multiple years across large regions.
* combine remotely sensed vegetation data with climate data. 
* map temporal trends in agriculturally-relevant climate metrics. 

### Setup

Create a new script in your *labs-gee/lab-8* repository called *indian-wheat-extreme-heat-trends.js*. Enter the following comment header to the script. 

```js
/*
Indian wheat extreme heat trends
Author: Test
Date: XX-XX-XXXX

*/

```

The assessment area for this lab will be croplands in two Indian states comprising the North-West Indo-Gangetic Plain: Haryana and Uttar Pradesh. Extract the boundaries for these states and centre the map on their location. 

```js
var aoi = ee.FeatureCollection('FAO/GAUL/2015/level1')
  .filter(ee.Filter.eq('ADM0_NAME', 'India'))
  .filter(ee.Filter.inList('ADM1_NAME',
      ['Punjab', 'Haryana']))
  .geometry();
 
Map.centerObject(aoi, 6);
Map.addLayer(aoi, {color: 'black'}, 'AOI', false);
```

Also set some config variables for this analysis.

```js
// Config
var startYear    = 2001;   
var endYear      = 2025;   // last complete season 
var EDD_THRESH   = 30;     // EDD temperature threshold, deg C
var WINDOW_DAYS  = 45;     
var VI_BAND      = 'NDVI'; // 'NDVI' or 'EVI' from MOD13Q1
var USE_CROPMASK = true;   // restrict outputs to cropland (ESA WorldCover)
var EXAMPLE_YEAR = 2015;   // a single season to visualise peak-VI timing
 
// Season for harvest-year Y = sown Nov (Y-1) -> harvested Apr (Y).
var SEASON_START_MONTH = 11; // November of year (Y-1)
var SEASON_END_MONTH   = 4;  // April of year (Y)
```

And, load the required datasets. The <a href="https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MOD13Q1" target="_blank">MODIS MOD13Q1 product</a> has a 16 day temporal resolution, a 250 m spatial resolution and provides NDVI and EVI bands. The <a href="https://developers.google.com/earth-engine/datasets/catalog/ECMWF_ERA5_LAND_DAILY_AGGR#description" target="_blank">ERA5-Land Daily Aggregated</a> dataset is a daily climate reanalysis product with an approximate 11 km spatial resolution. 

```js
// MODIS/Terra Vegetation Indices, 250 m, 16-day (Collection 6.1).
var MOD13 = ee.ImageCollection('MODIS/061/MOD13Q1').filterBounds(aoi);
 
// ERA5-Land daily aggregates (~11 km); 2 m air temperature in KELVIN.
var ERA5 = ee.ImageCollection('ECMWF/ERA5_LAND/DAILY_AGGR');
 
// Optional static cropland mask: ESA WorldCover v200, class 40 = Cropland.
var cropMask = ee.ImageCollection('ESA/WorldCover/v200').first().eq(40);
```

## Functions

One of the core approaches to working with spatio-temporal data in Google Earth Engine is mapping a function over an `ImageCollection` and performing analysis on `Image`s within temporal slices. The analysis operations are defined by the function. 

The following function can be mapped over an `ImageCollection` of MODIS MOD13Q1 `Image`s to identify the peak VI value and date of peak VI value per-pixel. Specifically, how this function works is:

1. for each year, extract all MODIS MOD13Q1 `Image`s that intesect the wheat growing season. 
2. for each `Image`, append a band for the day of year (that is what the variable `t` references).
3. return an `Image` with two bands: an NDVI value and a day of year.
4. apply the `qualityMosaic()` function to find the highest value per-pixel in an `ImageCollection`. This also carries through the day of year band so the `Image` that is returned contains a pixel-wise band for peak NDVI and a pixel-wise band for the date of peak NDVI. 

```js
// Map peak wheat growing season VI
function seasonPeak(Y) {
  Y = ee.Number(Y);
  var start = ee.Date.fromYMD(Y.subtract(1), SEASON_START_MONTH, 1);
  var end   = ee.Date.fromYMD(Y, SEASON_END_MONTH, 30);
 
  var stack = MOD13.filterDate(start, end).map(function(img) {
    var vi = img.select(VI_BAND).multiply(0.0001).rename('VImax');
    // composite start + 8 days ~= composite centre
    var t = ee.Image.constant(img.date().advance(8, 'day').millis())
              .toDouble().rename('peakMillis')
              .updateMask(vi.mask());
    return vi.addBands(t).clip(aoi);
  });
 
  // qualityMosaic picks, per pixel, the bands from the image with the highest
  // 'VImax'  ->  gives both the peak value and the time it occurred.
  return stack.qualityMosaic('VImax');
}
```

The next two functions compute the extreme heat metrics per-pixel in the temporal window following peak VI which corresponds to grain filling. 

`dailyTmaxExceedance(tmax, thr)` computes the extreme heat metric - a version of extreme degree days. Here, it sets a value of 0 to a daily max temperature of 30<sup>o</sup>C, a value of 1 to 31<sup>o</sup>C, a value of 2 to 32<sup>o</sup>C and so on.

The `seasonEDD()` function is more involved. It takes in a year and then calls the `seasonPeak()` function to get a map of peak wheat growing season NDVI for the year and the dates of peak NDVI. Then it iterates over each day within the wheat growing season, extracts the `temperature_2m_max` band, which is 2 m air temperature, computes the extreme heat metric, and then checks to see if the day in question falls within a pixel's grain filling period. Finally, it sums the extreme heat metric over the grain filling period to give a single number representing the cumulative extreme heat stress the crop was exposed to. 

```js
function dailyTmaxExceedance(tmax, thr) {
  return tmax.subtract(thr).max(0).rename('EDD');
}

function seasonEDD(Y) {
  Y = ee.Number(Y);
  var peakMillis = seasonPeak(Y).select('peakMillis');
  var winMillis  = ee.Number(WINDOW_DAYS).multiply(86400 * 1000);
 
  // Bracket all possible per-pixel windows. Peaks span ~mid-Jan to late Mar and
  // the window runs forward WINDOW_DAYS, so a late peak + 45 d reaches mid-May.
  var eStart = ee.Date.fromYMD(Y.subtract(1), 12, 1);
  var eEnd   = ee.Date.fromYMD(Y, 5, 31);
 
  var eddDaily = ERA5.filterDate(eStart, eEnd).map(function(day) {
    var tmax = day.select('temperature_2m_max').subtract(273.15);
    var edd  = dailyTmaxExceedance(tmax, EDD_THRESH);
 
    // Keep the day only if it falls in [peak, peak + WINDOW_DAYS].
    var dayMillis = ee.Image.constant(day.date().millis()).toDouble();
    var sincePeak = dayMillis.subtract(peakMillis);
    var inWindow  = sincePeak.gte(0).and(sincePeak.lte(winMillis));
 
    return edd.updateMask(inWindow);
  });
 
  return eddDaily.sum().rename('EDD')
    .set('year', Y)
    .set('system:time_start', ee.Date.fromYMD(Y, 3, 1).millis());
}

var years = ee.List.sequence(startYear, endYear);
var eddByYear = ee.ImageCollection(years.map(function(y) {
  return seasonEDD(y);
}));
```

## Mapping peak wheat growing season

Let's demonstrate the functions defined above with an example year. Explaining this:

`jan1   = ee.Date.fromYMD(EXAMPLE_YEAR, 1, 1).millis();` converts January 1<sup>st</sup> into date units and then into epoch milliseconds (the format Google Earth Engine uses for time).

Then `var peakDOY = peakEx.select('peakMillis').subtract(jan1).divide(86400000).rename('peakDOY');`:

* `peakEx.select('peakMillis')` is the image band holding, per pixel, the timestamp of maximum VI (also in epoch milliseconds).
* `.subtract(jan1)` gives the elapsed time from Jan 1 to each pixel's peak NDVI, in milliseconds. 
* `.divide(86400000)` converts milliseconds to days. 86400000 is one day in milliseconds (24 × 60 × 60 × 1000 = 86,400 s × 1000). So a peak on Jan 1 → ~0, Feb 20 → ~50, and so on.
* `.rename('peakDOY')` labels the resulting band.

```js
var peakEx = seasonPeak(EXAMPLE_YEAR);
var jan1   = ee.Date.fromYMD(EXAMPLE_YEAR, 1, 1).millis();
var peakDOY = peakEx.select('peakMillis').subtract(jan1)
    .divide(86400000).rename('peakDOY');
Map.addLayer(peakDOY.clip(aoi),
  {min: 30, max: 90, palette: ['00429d', '73a2c6', 'ffffbf', 'f4777f', '93003a']},
  'Peak VI day-of-year (' + EXAMPLE_YEAR + ')', false);
```

**Can you spot visual artefacts in the map of peak wheat growing season day of year? These maps are generated from 16 day MODIS NDVI composite images. Read up on composite images in Chapter 10 of Volume 2D in the <a href="https://www.eoa.org.au/earth-observation-textbooks" target="_blank">Earth Observation Australia textbooks</a>. Reflect on the strengths and weaknesses of using 16 day composite images to map peak wheat growing season date.**

<details>
  <summary><b>Can you generate a map of peak NDVI values for the <code>EXAMPLE_YEAR</code>?</b></summary>
<p>

```js
var peakNDVI = peakEx.select('VImax');

// Over to you for visualising this layer!
```
</p>
</details>


Now let's compute the peak NDVI, peak NDVI date, and extreme heat exposure during the wheat grain filling period for every pixel and year defined by the `startYear` and `endYear` variables. Start by creating a list of years (`var years = ee.List.sequence(startYear, endYear);`). Then map the function `seasonEDD()` that we defined previously over each year in the list. 

!!! note
    This is a good moment to make sure you are comfortable with the concept of mapping functions over collections in Google Earth Engine. It is one of the key features of the Google Earth Engine platform and lets you scale spatio-temporal analysis across large areas and big geospatial datasets. Review the docs <a href="https://developers.google.com/earth-engine/guides/ic_mapping" target="_blank">here</a>.

```js
var years = ee.List.sequence(startYear, endYear);
var eddByYear = ee.ImageCollection(years.map(function(y) {
  return seasonEDD(y);
}));

// Compute an overall mean EDD layer for visualisation
var meanEDD = eddByYear.mean().rename('EDD_mean');
```

## Trend analysis

The data is now prepared for computing temporal trends in extreme heat during the wheat crop's grain filling period. To be specific, the annual increase in extreme heat exposure during the grain filling period will be computed per-pixel. 

<details>
  <summary><b>Why are we dynamically computing a pixel and year specific grain filling period?</b></summary>
<p>

The study area is quite large spanning a substantial portion of North-West India and the Ganges Basin. Climatic and farm managment conditions vary across this extent so hard coding a grain filling period would likely mischaracterise the key period of wheat crop vulnerability to extreme heat in some locations. Further, the rice and wheat crops are grown in rotation. The sowing and growing season dates are affected by the harvest time of the preceding rice crop which introduces year-to-year variability. 
</p>
</details>

<details>
  <summary><b>Why does dynamically computing a pixel and year specific grain filling period mean we need to be careful about interpreting trends in the crop's extreme heat exposure?</b></summary>
<p>

An observed trend in extreme heat exposure might not be due to a climatic trend (i.e. an increase in extreme heat events over time) but could be due to changes in growing season dates over time (i.e. a trend of sowing crops later pushes the grain filling stage into warmer parts of the year). 
</p>
</details>

Google Earth Engine provides a `linearFit()` reducer for computing trends per-pixel using ordinary least squares regression. However, to use this reducer the data needs `ImageCollection` where each `Image` has a time band and a band storing the variable to compute trends for. 

The below code snippet `map`s over the `eddByYear` `ImageCollection` and extracts the band storing the per-pixel extreme heat metric and adds a year band. Conceptually, the year band will map onto the X-axis and and the `EDD` band will map onto the Y-axis in a trend analysis. 

Calling `.reduce(ee.Reducer.linearFit())` will compute slope and intercept coefficients per-pixel using the `year` and `EDD` bands. This returns an `Image` with two bands: `scale` and `offset`. `scale` corresponds to the slope coefficient in each pixel which is the trend of the annual increase in extreme heat exposure per year. 

```js
var trend = eddByYear.map(function(img) {
    var yr = ee.Image.constant(ee.Number(img.get('year'))).toDouble().rename('year');
    return yr.addBands(img.select('EDD')).select(['year', 'EDD']);
});
 
// Ordinary least squares trend
var slopeOLS = trend.reduce(ee.Reducer.linearFit())
    .select('scale').rename('EDD_trend_OLS');   // deg C-day / yr
```

<details>
  <summary><b>At the top of the script a cropland mask <code>cropMask</code> was loaded. Can you clip this mask to the <code>aoi</code> and mask out non-cropland areas from the <code>slopeOLS</code> <code>Image</code>? Call this <code>Image</code> <code>slopeOLSCrops</code>.</b></summary>
<p>

```js
var slopeOLSCrops = slopeOLS.updateMask(cropMask);
```
</p>
</details>

**Can you map the `slopeOLSCrops` `Image` to visualise spatial patterns in trends of extreme heat exposure during the wheat crop's grain filling stage? Think carefully about how you visualise this data. You are mapping a trend that can be positive (increasing extreme heat exposure) or negative (decreasing extreme heat exposure). What colour palettes help interpretation of these diverging signals?**. 

**What is you initial assessment of the trend of the wheat crop's exposure to extreme heat during grain filling in the North-West Indo-Gangetic Plain?**

<details>
  <summary><b>What could the cause of negative trends in extreme heat exposure?</b></summary>
<p>
Inter-annual variation in extreme heat exposure with some "cooler" years towards the end of the time-series pulling the trend down. Or, it could be a signal of farmer adaptation shifting planting dates earlier to avoid periods of extreme heat. This could be explored by i) comparing trends in extreme heat exposure during a fixed temporal window (e.g. March and April) - this would identify if there's a genuine climatic trend, or ii) looking for trends in start-of-season date and seeing if there is a correlation between trends in earlier start-of-season (proxy for planting) and negative trends in grain filling extreme heat exposure. 
</p>
</details>

<details>
  <summary><b>The trends on the map have a 250 m spatial resolution corresponding to MODIS pixels. Why does this require careful interpretation when MODIS data is combined with ~11 km sptial climate data?</b></summary>
<p>
The finer spatial resolution detail on the map all comes from the MODIS satellite images and the pixel-to-pixel variation in phenology. It does not reflect locally varying trends in extreme heat at sub 11 km spatial resolution scales.
</p>
</details>

Climatic data is often affected by extreme values or years which exert large influence on trends estimated using ordinary least squares regression. An alternative is to compute Sen's slope - the median difference between all pairs of points in a time-series. 

<details>
  <summary><b>This is the documentation for the <a href="https://developers.google.com/earth-engine/apidocs/ee-reducer-sensslope" target="_blank">Sen's Slope</a> in Google Earth Engine? Can you compute the trend in extreme heat exposure using Sen's slope and map the trends using the same visualisation parameters that were applied to the OLS trends?</b></summary>
<p>

```js
var slopeSen = trend.reduce(ee.Reducer.sensSlope())
    .select('slope').rename('EDD_trend_Sen');   // deg C-day / yr
var slopeSenCrops = slopeSen.updateMask(cropMask);

// Over to you for the visualisation
```
</p>
</details>

**Homework activities**

Trends in extreme heat exposure have been computed for multiple pixels. However, we don't know where these trends are statistically significant. To ascertain how much more vulnerable North-West India's wheat crop is becoming to extreme heat we need to perform significance testing and mask out pixels with "non-significant" slopes. There is an example of how to do this using Sen's Slope <a href="https://developers.google.com/earth-engine/tutorials/community/nonparametric-trends#significance_testing" target="_blank">here</a>. Can you adapt this to perform significance testing for `slopeSenCrops`. 

What is the area of Punjab and Haryana that is affected by statistically significant trends of increasing extreme heat exposure during the grain filling stage? What is your assessment of how vulnerable the wheat crop is to extreme heat in this region? 

This next activity tests how well you can adapt this workflow to a different problem. Farm dams in Western Australia are a critical non-potable on-farm water resource in a dryland cropping landscape. The farm dams are filled by rainfall and runoff from bare earth "roaded" catchments. To generate runoff roaded catchments typically need rainfall events of at least 8 mm. Can you:

1) find a dataset storing daily precipitation in Google Earth Engine's data catalog. 

2) compute the number of rainfall events greater than 8 mm per year.

3) map the trends across South-West Western Australia in the number of 8 mm rainfall events. per year. 