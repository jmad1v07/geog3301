# Australian 2019/2020 bush fires - intro to time-series data

Towards the end of 2019 and in early 2020 Australia was impacted by widespread and severe bushfires. These bushfires burnt through vast areas of vegetation, caused habitat losses and animal deaths and affected air quality over urban areas. This lab will demonstrate how multi-temporal remote sensing data can be used to map the bushfire's effect on urban air quality over Sydney.

The <a href="https://sentiwiki.copernicus.eu/web/s5p-mission" target="_blank">Sentinel-5P (Sentinel-5 Precursor Mission)</a> comprises a satellite carrying the TROPOMI instrument which senses reflectance in the UV, visible, near infrared and shortwave infrared wavebands. It is designed for high spatio-temporal atmospheric assessments of air quality, pollution and ozone. Sentinel-5P has a 16-day repeat orbit, but due to its wide swath it provides daily global coverage at 13:30 time. This lab will use data captured by Sentinel-5P to identify the effects of the 2019-2020 Australian bushfires on urban air quality. 

<figure markdown="span">
  ![Sentinel-5P](https://sentiwiki.copernicus.eu/__attachments/a_33032c6b8d86f5f2902ff34345b5c9905d81d1260f5f87e7f75fa68c708aacb0/Copernicus_Sentinel-5P_Europe_s_air_quality_monitoring_mission.png)
  <figcaption>Sentinel-5P satellite (Source: ESA).</figcaption>
</figure>


This lab will focus on developing the following skills:

* filtering `ImageCollection`s of remote sensing data to conduct pre- and during-event assessments.
* charting time-series of remote sensing-derived variables.
* searching for data in the Google Earth Engine data catalog.
* using reference data to interpret patterns in temporal data. 
* adapting Google Earth Engine workflows for different spatial analysis problems.

### Setup

Create a new script in your *labs-gee/lab-8* repository called *australian-bushfires-time-series.js*. Enter the following comment header to the script. Start by adding a point over Sydney and centre the map display accordingly. 

```js
/*
Australian Bushfire - time-series skills 
Author: Test
Date: XX-XX-XXXX

*/

var aoiPoint = ee.Geometry.Point([151.15762986467942, -33.841619007043796]);
Map.addLayer (aoiPoint);
Map.centerObject(aoiPoint, 6);
```

It is good practice to make scripts modular. Often geospatial and remote sensing analysis workflows have common and repeating operations and it is inefficient to regenerate very similar scripts for different tasks. One way of making scripts modular is to add all the key variables and constants for a given analysis to the top of a script and then place the data processing, analysis and visualisation operations below. If you want to reuse the script's logic or functionalities, all you need to do is change the variables and constants up front. 

There are two variables below that have been left blank: `s5RawImColl` and `s5band`. There are a variety of Sentinel-5P `ImageCollection`s that you can access from the <a href="https://developers.google.com/earth-engine/datasets/catalog/sentinel-5p" target="_blank">Google Earth Engine Data Catalog</a>. **Identify which Sentinel-5P product would be most suited to assessing bushfire's effect on urban air quality over Sydney. Use the informaiton provided in the Google Earth Engine data catalog to fill in the empty variables.**

```js
// Config

// Geometries for different assessment areas
var aoiPoly = aoiPoint.buffer(500000);
var aoiUrban = aoiPoint.buffer(20000);

// Dates for the pre- and during-bushfire event assessment
var preStart = '2018-12-01';
var preEnd = '2018-12-31';
var duringStart = '2019-12-01';
var duringEnd = '2019-12-31';


// Identifiers for the Sentinel-5 product
var s5ImColl = '';
var s5band = '';
```

## Sentinel-5P data

The Sentinel-5P data is stored as an `ImageCollection` - multiple `Image` objects for different dates and locations. This needs to be filtered to the study area surrounding Sydney referenced by `aoiPoly` and to retain only the band of interest (i.e. the band you have selected that represents air quality).

<details>
  <summary><b>Using the config variables for <code>aoiPoly</code>, <code>s5ImColl</code>, <code>s5band</code>, filter the Sentinel-5P <code>ImageCollection</code>. Call this <code>ImageCollection</code> <code>s5</code></b></summary>
  <p>

```js
var s5 = ee.ImageCollection(s5ImColl)
    .filterBounds(aoiPoly)
    .select(s5band);
```
</p>
</details>

## Pre- and during-bushfire urban air quality assessment

To identify the effect of bushfire on air quality over Sydney, pre- and during-bushfire air quality `Image`s need to be compared. This requires filtering pre- and during-event Sentinel-5P `Image`s from the `s5` `ImageCollection` and creating composite pre- and during-event `Image`s for comparison. 

<details>
  <summary><b>Using the config variables for <code>preStart</code>, <code>preEnd</code>, <code>duringStart</code>, <code>duringEnd</code> can you filter <code>s5</code> and create mean composite <code>Image</code>s. Call these <code>Image</code>s <code>s5Pre</code> and <code>s5during</code>. Clip both of these composite <code>Image</code>s to the extent of <code>aoiPoly</code>.</b></summary>
  <p>

```js
var s5Pre = s5
  .filterDate(preStart, preEnd)
  .mean()
  .clip(aoiPoly);

var s5during = s5
  .filterDate(duringStart, duringEnd)
  .mean()
  .clip(aoiPoly);
```
</p>
</details>

If you have successfully created `s5Pre` and `s5during`, the following code snippet should visualise both layers on the map display. Toggle the layer's opacity sliders to visually see the difference in air quality between the pre- and during-event assessment periods. 

```js
// Visualize parameters.
var s5Viz = {
    min: -1,
    max: 2,
    palette: ['black', 'blue', 'purple', 'cyan', 'green',
        'yellow', 'red'
    ]
};

Map.addLayer(s5Pre, s5Viz, "Pre image");
Map.addLayer(s5during, s5Viz, "During image");
```

<details>
  <summary><b>Focusing on Sydney's urban area, can you create a change <code>Image</code> showing the difference in air quality between the two dates and visualise it on the map?</b></summary>
  <p>

```js
var s5change = s5during.subtract(s5Pre);

// Over to you for visualising this layer!
```
</p>
</details>

## Air quality time-series

The following code snippet demonstrates how to plot daily time-series for two groups. Here, these two groups are the pre- and during-event months and the focus is on comparing urban air quality, so use the `aoiUrban` geoemtry. 

```js
// Create a function to get the mean S5 variable
function getVar(collectionLabel, img) {
    return function(img) {
        // Calculate the mean S5 variable within the target geometry.
        var s5Mean = img.reduceRegion({
            reducer: ee.Reducer.mean(),
            geometry: aoiUrban,
            scale: 7000
        }).get(s5band);

        // Get the day-of-year of the image.
        var doy = img.date().getRelative('day', 'year');

        // Return a feature with s5 product mean and day-of-year properties.
        return ee.Feature(null, {
            's5var': s5Mean,
            'DOY': doy,
            'type': collectionLabel
        });
    };
}

// Get the S5 variables for a pre and during collection 
// and merge for plotting.
var s5Change_forPlotting = s5
    .filterDate(duringStart, duringEnd)
    .map(getVar('during'))
    .merge(s5.filterDate(preStart, preEnd)
    .map(getVar('pre')));
s5Change_forPlotting = s5Change_forPlotting
    .filter(ee.Filter.notNull(['s5var']));

// Plot the chart.
var preDuringChart = ui.Chart.feature.groups(
        s5Change_forPlotting, 'DOY', 's5var', 'type')
    .setChartType('LineChart')
    .setOptions({
        title: 'DOY time series for mean ' + s5band +
            ' for pre and during assessment periods'
    });

print(preDuringChart);
```

The chart should clearly show the difference in air quality over Sydney between the pre- and during-bushfire event periods. There should be more days during the bushfire with worse air quality. 

<details>
  <summary><b>The bushfires affected Australia for a longer period than December 2019. To assess the duration of the period Sydney's residents were exposed to lower air quality can you generate a single longer-term time-series from <code>'2018-12-01'</code> to <code>'2020-05-01'</code>?</b></summary>
  <p>

```js
// Over to you for doing this! 
// Generating time-series from reductions of ImageCollections is a common remote sensing task
// Having code written to do this will be frequently useful.
```
</p>
</details>

## Verification and interpretation

A key part of all remote sensing analysis is verifying the results and using reference data to interpret patterns in images. Thus far the analysis has shown a clear pattern of lower air quality over Sydney during a period when bushfires were burning compared to the same month in the previous year. The inference that this pattern of lower air quality is associated with the bushfires hinges on our selection of the dates for the assessment periods. However, further analysis should be undertaken to link the change in air quality over Sydney to bushfires. One way to do this is map out where fires were burning during this period and see if the spatial features of worsening air quality on the map display correlate with fire locations. 

<details>
  <summary><b>Using the <a href="https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MOD14A1#bands" target="_blank">MODIS MOD14A1.061: Terra Thermal Anomalies & Fire Daily Global 1km</a> product, can you create a map of the extent of all fires that occurred during the December 2019? This should be a binary map where pixels equal 1 if a fire was detected at any time during that month and <code>null</code> if the pixel was unburnt. Colour the burnt extent map with a suitably bright colour and overlay it on top of the during event air quality maps.</b></summary>
  <p>

Start with the <a href="https://developers.google.com/earth-engine/datasets/catalog/MODIS_061_MOD14A1#bands" target="_blank">MODIS MOD14A1.061: Terra Thermal Anomalies & Fire Daily Global 1km</a> and look at the <code>FireMask</code> and the Fire (high confidence, land or water) bit. 
</p>
</details>

**Homework activity**

Using Landsat data and differencing spectral indices, can you create a burnt extent map around Sydney. Does it align with your map of MODIS fire detections?

Can you edit this script and apply it to the Wooroloo Bushfire near Perth in 2021?
