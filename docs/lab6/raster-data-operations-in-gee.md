# Raster data operations in Google Earth Engine

## Introduction

This lab will introduce data transformation and manipulation operations for raster data.

## Quick recap

In Google Earth Engine, raster data is represented using `Image` objects which include one or more bands and each band is a georeferenced raster. Each band has `properties` such as data type (e.g. integer), scale (spatial resolution), band name and projection. 

The `Image` object itself contains metadata relevant to all bands inside a `properties` dictionary object (e.g. date of `Image` capture). 

A collection of `Image`s are stored in an `ImageCollection`. For example, all Landsat 8 `Image`s are stored in an `ImageCollection` object.

## Data transformation

Data transformation operations transform data into a format ready for subsequent analysis. Data transformation operations can be spatial (manipulating geometries or raster pixels) or non-spatial (manipulating attributes). 

Raster data transformation operations can be spatial (e.g. changing the spatial resolution of raster pixels, clipping a raster extent), non-spatial and applied to metadata (e.g. filtering raster `Image`s by date), or applied to values stored in raster pixels (e.g. applying a function to convert raster `Image` pixel values that represent temperature in Fahrenheit to Celsius).

Data transformation operations applied to raster data can also be categorised as:

1. <b>local operations:</b> a pixel value is computed using per-pixel arithmetic, comparison, or logical operations.
2. <b>focal operations:</b> a pixel value is computed using aggregation or summary operations applied to neighbouring pixel values defined by a moving window or kernel. 
3. <b>zonal operations:</b> a pixel value is computed using aggregation or summary operations applied to pixels within an irregular region.
4. <b>global operations:</b> an output is the result of aggregation or summary operations applied to all pixels in a raster data set. 

### Where has vegetation cover changed between 2000-2002 and 2017-2019?

This lab demonstrates a range of raster data transformations to address the question: *Where has vegetation cover changed between 2000-2002 and 2017-2019?* The study area is the urban Perth and Peel region.

To address this question you will be using surface reflectance data from the Landsat 7 and Landsat 8 satellites. 

The Landsat 7 surface reflectance data recorded by the Enhanced Thematic Mapper (ETM+) sensor which has a revisit period of 16 days and a 30 m spatial resolution. Data from Landsat 7 is available from 1999 in Google Earth Engine. 

The Landsat 8 surface reflectance data is recorded by the Operational Land Imager (OLI) sensor. Again, Landsat 8 data has a revisit period of 16 days and a spatial resolution of 30 m. Landsat 8 observations are available from 2013. 

The following are good resources for information about Landsat data:

* <a href="https://esajournals.onlinelibrary.wiley.com/doi/full/10.1002/ecy.1730" target="_blank">A survival guide to Landsat preprocessing</a>
* <a href="https://www.sciencedirect.com/science/article/pii/S003442571400042X" target="_blank">Landsat-8: Science and product vision for terrestrial global change research</a>
* <a href="" target="_blank">Benefits of the free and open Landsat data policy</a>
* <a href="https://www.sciencedirect.com/science/article/pii/S0034425719300707" target="_blank">Current status of Landsat program, science, and applications</a>

This lab will also introduce you to Google Earth Engine's capacity to process big geospatial data sets. Here, you will use all Landsat 7 and 8 scenes that intersect with the Perth and Peel region between 2000 and 2002 and 2017 and 2019. 

Your workflow will encompass the following steps:

1. Subset all Landsat 7 and 8 `Image`s that intersect with study region. 
2. Mask cloudy pixels out of all Landsat 7 and 8 `Image`s returned from step 1. 
3. Merge Landsat 7 and Landsat 8 `Image`s into one `ImageCollection`.
4. Compute the NDVI for all cloud free Landsat pixels between 2000-2002 and 2017-2019.
5. Compute a median NDVI composite `Image` for 2000-2002 and 2017-2019.
6. Compute a difference `Image` showing change in NDVI between 2000-2002 and 2017-2019.

### Setup

Create a new script in your *labs-gee/lab-6* repository called *raster-data-operations-in-gee.js*. Enter the following comment header to the script. 

```js
/*
Raster data operations
Author: Test
Date: XX-XX-XXXX

*/

// import data

// import Landsat 7 and 8 ImageCollections
var l7 = ee.ImageCollection('LANDSAT/LE07/C02/T1_L2');
var l8 = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2');

var bBox = ee.Geometry.Polygon(
        [[[115.73339493500093, -32.00885297734888],
          [115.73339493500093, -32.17638807304199],
          [115.97097428070406, -32.17638807304199],
          [115.97097428070406, -32.00885297734888]]], null, false);

```

## Filter

`ImageCollection`s can be filtered using spatial and non-spatial filters. The `filterBounds()` function filters  `Image`s in an `ImageCollection` that intersect with the extent of a `Geometry` object (passed as an argument to the `filterBounds()` function). This is a spatial filtering operation. 

`ImageCollection`s can also be filtered using non-spatial operations using the values of properties in `Image` metadata. Each Landsat `Image` has a `properties` dictionary object storing metadata attributes including the date and time of image capture (the `system:time_start` property). This property can be used with the `filterDate()` function to subset all Landsat `Image`s captured within a specified date range. String data for the start and end date of the filterting period are passed as arguments into the `filterDate()` function and an `ImageCollection` containing only `Image`s captured during that period is returned. 

Executing the code below will filter the Landsat 8 and Landsat 7 `ImageCollection`s using the `Geometry` object in `bBox` and subset `Image`s captured between the dates specified in the `filterBounds()` function.

```js
/* filter Landsat 7 and 8 ImageCollections to return only Images 
that intersect the bBox and were captured within specified date range */
var l72000 = l7
  .filterBounds(bBox)
  .filterDate('2000-01-01', '2002-12-31');
  
var l72017 = l7
  .filterBounds(bBox)
  .filterDate('2017-01-01', '2019-12-31');
  
var l82017 = l8
  .filterBounds(bBox)
  .filterDate('2017-01-01', '2019-12-31');

print('Landsat 7 Im Coll 2000-2002:', l72000);

```

Print the Landsat 7 `ImageCollection` that is returned from filtering the Landsat 7 archice `ImageCollection` for `Images` that intersect with the `Geometry` object `bBox` and were captured between 1 January 2000 and 31 December 2002. 

<figure markdown="span">
  ![Print() of filtered Landsat 7 ImageCollection.](../images/l7-filtered-imcoll.png)
  <figcaption>`Print()` of filtered Landsat 7 `ImageCollection`.</figcaption>
</figure>

<details>
  <summary><b>What would happen if you passed the dates <code>'2010-01-01', '2010-12-31'</code> into the `filterBounds()` for a Landsat 8 `ImageCollection`?</b></summary>
  <p>
  Error message - there should be no Landsat 8 `Image`s captured in 2010. Landsat 8 started collecting data in 2013. 
  </p>
</details>


### Cloud Masks

You have just applied spatial and non-spatial filtering operations to an `ImageCollection`. You can also apply filtering operations to an individual `Image`. A common application of spatial filtering operations applied to an `Image` in remote sensing is masking out a pixel value based on the pixel value in the corresponding location in another raster. 

Remote sensing data products often have a pixel quality band which indicates if the observation for a pixel is high quality or not. Cloud cover and atmospheric contamination are common sources of low quality observations. 

The following code block displays an `Image` from the filtered Landsat 8 `ImageCollection` `l82017` as an RGB composite. Cloud cover contamination is clearly visible. 

```js
// cloud mask function

// visualise a cloudy Landsat Image
var cloudyL8 = ee.Image('LANDSAT/LC08/C02/T1_L2/LC08_112082_20170510');

/* Define the visualization parameters. The bands option allows us to specify which bands to map. 
Here, we choose B4 (Red), B3 (Green), B2 (Blue) to make a RGB composite image.*/ 
var vizParams = {
  bands: ['SR_B4', 'SR_B3', 'SR_B2'],
  min: 0,
  max: 20000,
};
Map.centerObject(cloudyL8, 8);
Map.addLayer(cloudyL8, vizParams, 'Cloudy L8 Image');

```

<figure markdown="span">
  ![Cloud contamination of Landsat 8 Image.](../images/cloudy-l8-image.png)
  <figcaption>Cloud contamination of Landsat 8 `Image`.</figcaption>
</figure>

Landsat surface reflectance pixel quality attributes are stored as a bitmask within the `QA_PIXEL` band generated by the <a href="https://www.usgs.gov/land-resources/nli/landsat/cfmask-algorithm" target="_blank">CFMASK algorithm</a>. Other remote sensing products (e.g. MODIS and Planet) also provide pixel quality information as a bitmask; therefore, bitmasks are an important concept to understand when working with remote sensing data.

Each pixel in a quality band stores a number which can be represented as a decimal number (e.g. 32) or as a binary number (e.g. 00100000 - this is an 8 bit integer or one byte). 

<b>Binary numbers</b>

* each bit in the binary number can take a value of 0 or 1 
* bits are ordered from right to left (i.e. 00001001 - bit 0 and bit 3 are represented by the binary digit 1)
* bit order starts at 0 (i.e. bit 5 is set to 1 here - 00100000) 
* binary numbers have base 2 
* convert binary 00001001 to decimal &rarr; 00001001 = $(0*2^7) + (0*2^6) + (0*2^5) + (0*2^4) + (1*2^3) + (0*2^2) + (0*2^1) + (1*2^0)$ = 9


In a bitmask `Image`, each bit corresponds to an indicator of quality information for that pixel. Keeping with the binary number 00100000, the bitmask represented by bit 5 evaluates to true and all other bitmasks are false. The bitmasks represented by values in the `QA_PIXEL` band of Landsat `Image`s are shown below. The `QA_PIXEL` band stores pixel values as unsigned 16 bit integers with bits 0 to 10 representing bitmasks forxs different aspects of pixel quality. 

This <a href="https://gemini.google.com/share/eca605915134?skid=40457113-45fb-4290-acb6-5fdcba26ab09" target="_blank">web app</a> provides an interactive explainer of how bitmasks work.

Look at the <a href="https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LC08_C02_T1_L2#bands" target="_blank">docs</a> for the `QA_PIXEL` band you will see the first 5 bits correspond to cloudy pixels that should be masked.

A high quality pixel observation would have the digit 0 for the first five bits. A cloud pixel would have the value one in bit 3 (i.e. 0000000000001000). While using bitmasks to store pixel quality information is more complicated than using separate binary `Image` bands for each indicator of pixel quality, it is a more efficient way of storing and transporting this information.  

<figure markdown="span">
  ![QA_PIXEL bitmask derived from CFMASK for Landsat surface reflectance.](../images/cf_mask_pixel_qa.png)
  <figcaption>`QA_PIXEL` bitmask derived from CFMASK for Landsat surface reflectance.</figcaption>
</figure>

<details>
  <summary><b>If a pixel was clear and snow what would its <code>QA_PIXEL</code> value be in binary?</b></summary>
  <p>
  0000000001100000 (bit 5 for snow and bit 6 for clear)
  </p>
</details>

You can use the bitmask contained in the `QA_PIXEL` band to mask out cloudy pixels. To do this you need identify which pixels have the digit 1 in bits 0 to 4 (i.e. they are cloudy) and then mask those pixels in the Landsat 8 `Image`. The following steps demonstrate how to do this: 

**1** `image.select('QA_PIXEL')` extracts the `QA_PIXEL` band from the Landsat image. A 16 bit unsigned integer raster. 

**2** `parseInt('11111', 2)` is evaluated in your browser and converts the string into an integer (31). When this integer value is written in binary / base-2 format it has the digit 1 in the first five bits (0000000000011111). 

**3** `bitwiseAnd(31)` compares the interger value 31 to the pixel values of the `QA_PIXEL` band on a bit by bit basis. It evaluates to 1 if the corresponding bits in both integers have the value 1, otherwise it evaluates to 0. The comparison integer 31 representing the digit 1 in the first five bits is compared bitwise to each pixel in the `QA_PIXEL` band and if there are 1s in the corresponding bits then it indicates the pixel is cloudy.

<figure markdown="span">
  ![bitwiseAnd.](../images/bitwiseAnd.png)
  <figcaption>Example of applying the <code>bitwiseAnd()</code> operator to `QA_PIXEL` and cloud integer.</figcaption>
</figure>

**4** `.eq(0)` this will evaluate to True is the pixel is cloud free as there will be no cases where there is a 1 digit in the first five bits of `QA_PIXEL`. 

<table style="width:100%; border-collapse: collapse; border-bottom: 1px solid #ddd; padding: 15px;">
  <caption><code>bitwiseAnd</code> operation.</caption>
  <tr>
    <th>Situation</th>
    <th>Operation</th>
    <th>Operand</th>
    <th>Output</th>
  </tr>
  <tr>
    <td>cloud shadow present in <code>QA_PIXEL</code></td>
    <td><code>pixelQA.bitwiseAnd(shadowBitMask)</code></td>
    <td>0000000001010000 & 0000000000010000</td>
    <td>0000000000010000</td>
  </tr>
  <tr>
    <td>cloud shadow NOT present in <code>QA_PIXEL</code></td>
    <td><code>pixelQA.bitwiseAnd(shadowBitMask)</code></td>
    <td>0000000001000000 & 0000000000010000</td>
    <td>0000000000000000</td>
  </tr>
   <tr>
    <td>cloud present in <code>QA_PIXEL</code></td>
    <td><code>pixelQA.bitwiseAnd(cloudsBitMask)</code></td>
    <td>0000000000011000 & 0000000000001000</td>
    <td>0000000000001000</td>
  </tr>
</table>


If the result of applying a `bitwiseAnd` operation to a pixel value in `QA_PIXEL` and either `parseInt('11111', 2)` is not equal to zero then the bitmask indicates the pixel is cloudy and it should be masked from subsequent processing. 

The following code puts all these commands together. `cloudMask` stores a raster `Image` with a pixel value of 1 indicating no cloud and a pixel value 0 indicating cloud. When visualised on the map, cloud pixels should render in black.



```js
// make a cloud mask
var cloudMask = cloudyL8.select('QA_PIXEL').bitwiseAnd(parseInt('11111', 2)).eq(0);
Map.addLayer(cloudMask, {}, 'cloud mask');
```

<figure markdown="span">
  ![cloudMask Image derived from the QA_PIXEL bitmask.](../images/cloud-mask.png)
  <figcaption>`cloudMask` `Image` derived from the `QA_PIXEL` bitmask.</figcaption>
</figure>

You can use the `Image` stored in the variable `cloudMask` to mask cloudy pixel values in the Landsat 8 `Image` `cloudyL8`. <a href="https://developers.google.com/earth-engine/tutorials/tutorial_api_05#masking" target="_blank">Masking</a> pixels in Google Earth Engine makes them transparent and removes them from subsequent processing. To mask `Image` pixel values in Google Earth Engine pass a raster `Image`, where pixel values of zero indicate locations to mask, into the `updateMask()` function. 

```js
// mask out clouds in the cloudyL8 image
var cloudyL8Mask = cloudyL8.updateMask(cloudMask);
Map.addLayer(cloudyL8Mask, vizParams, 'Cloud Masked L8 Image');

```

<figure markdown="span">
  ![cloudMask applied to Landsat 8 Image.](../images/cloud-masked-l8.png)
  <figcaption>`cloudMask` applied to Landsat 8 `Image`.</figcaption>
</figure>

Use the <b>Layers</b> widget to toggle the unmasked and masked Landsat 8 `Image` on and off to see the effect of cloud masking. 

You have gone through the process of masking out cloudy pixels from a single Landsat 8 `Image`. However, repeating this process manually for all Landsat 8 `Image`s would be time consuming. This is where you can take advantage of a programmatic approach to GIS. You can wrap up the steps to create and apply cloud masks in a function. You can then `map` that function over an `ImageCollection` of Landsat `Image`s masking out cloudy pixels in each `Image`. 

```js
// Function to mask clouds based on the QA_PIXEL band of Landsat data.
function cloudMaskFunc(image) {
  var cloudMask = image.select('QA_PIXEL').bitwiseAnd(parseInt('11111', 2)).eq(0);
  return image.updateMask(cloudMask);
}
```

Next, you need to `map` this function over each of your filtered `ImageCollection`s of Landsat 7 and 8 data.

```js
// map cloudMask function over Landsat ImageCollections.
l72000 = l72000.map(cloudMaskFunc);

l72017 = l72017.map(cloudMaskFunc);

l82017 = l82017.map(cloudMaskFunc);

```

The function `cloudMaskFunc` is mapped over the `ImageCollection`s `l72000`, `l72017`, and `l82017`, each `Image` in these `ImageCollection`s is cloud masked, and an `ImageCollection` of cloud masked `Image`s is returned. 

<figure markdown="span">
  ![Graphical representation of mapping a function over a collection and returning a collection as an output (source: Wickham (2020)).](../images/function-map.png)
  <figcaption>Graphical representation of mapping a function over a collection and returning a collection as an output (source: [Wickham (2020)](https://adv-r.hadley.nz/index.html)).</figcaption>
</figure>

The concept of masking pixel values based on a cloud mask can be considered spatial subsetting; you are subsetting pixels for subsequent analysis based on their pixel values and their location in another raster `Image` <a href="https://geocompr.robinlovelace.net/spatial-operations.html#spatial-raster-subsetting" target="_blank">(Lovelace et al. 2020)</a>. This is also an example of a local raster operation where operations are applied on a per-pixel basis.

## Create new variables

### Local map algebra operations

Local operations are applied to `Image`s on a per-pixel basis. The output from a local operation is a raster where each pixel's value is the result of the local operation. The input to a local operation is either:

1. two or more `Image`s where the operation is applied on a per-pixel basis.
2. one or more `Image`s and a constant number where the constant number is combined with each pixel value via a mathematical operator. 

<figure markdown="span">
  ![Illustration of local operations with two or more Images where the operation is applied on a per-pixel basis - output = R1 + R2 (source: Gimond (2019)).](../images/map-algebra-image-image.png)
  <figcaption>Illustration of local operations with two or more <code>Image</code>s where the operation is applied on a per-pixel basis - <code>output = R1 + R2</code> (source: <a href="https://mgimond.github.io/Spatial/index.html" target="_blank">Gimond (2019)</a>).</figcaption>
</figure>

<figure markdown="span">
  ![Illustration of Image math with two or more Images and a constant number where the constant number is combined with each pixel value using the specified math operator - output = 2 * + raster + 1 (source: Gimond (2019)).](../images/map-algebra-image-constant.png)
  <figcaption>Illustration of <code>Image</code> math with two or more <code>Image</code>s and a constant number where the constant number is combined with each pixel value using the specified math operator - <code>output = 2 * + raster + 1</code> (source: <a href="https://mgimond.github.io/Spatial/index.html" target="_blank">Gimond (2019)</a>).</figcaption>
</figure>

Pixel values across `Image`s or pixel values and constant numbers can be combined using math operators in Google Earth Engine:

* `add()`
* `subtract()`
* `multiply()`
* `divide()`

Per-pixel comparison and logical operators that evaluate to true or false can also be used.

* `lt()` - less than
* `gt()` - greater than
* `lte()` - less than or equal to
* `gte()` - greater than or equal to
* `eq()` - equal to
* `neq()` - not equal to
* `and()` - AND

In Google Earth Engine only the intersection of unmasked pixels between input `Image`s are returned from local operations. 

### Rescaling

Landsat surface reflectance images are stored in integer format and to convert them back to surface reflectance units you need to apply scaling factors via local operations to each pixel. 

You can find the scaling factors for <a href="https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LE07_C02_T1_L2" target="_blank">Landsat 7</a> and <a href="https://developers.google.com/earth-engine/datasets/catalog/LANDSAT_LC08_C02_T1_L2#bands" target="_blank">Landsat 8</a> in the scale and offset columns. 

You can create a function apply the scaling factors to an `Image`. Note how the `multiply()` and `add()` operators combine a constant with every pixel value. 

Map this function over your Landsat `ImageCollection`s to convert them into surface reflectance units.

```js
function rescaleLandsat(image) {
  // Apply the scaling factors to the appropriate bands.
  var opticalBands = image.select('SR_B.').multiply(0.0000275).add(-0.2);
  var thermalBands = image.select('ST_B.*').multiply(0.00341802).add(149.0);

  // Replace the original bands with the scaled ones and apply the masks.
  return image.addBands(opticalBands, null, true)
      .addBands(thermalBands, null, true);
}

// convert to surface reflectance
l72000 = l72000.map(rescaleLandsat);

l72017 = l72017.map(rescaleLandsat);

l82017 = l82017.map(rescaleLandsat);

```

### Spectral indices

Mathematical per-pixel combinations of spectral reflectance measures are called spectral indices. Different land surface features have different spectral signatures (they reflect differently across wavelengths of the electromagnetic spectrum). Spectral indices combine information about levels of reflectance in different wavelengths into a single value. Computing spectral indices can provide more information about the characteristics of a pixel (location on the Earth's lands surface) than could be obtained from spectral reflectance measures in a single band. 

Spectral indices are commonly used to monitor vegetation (vegetation indices). The normalised difference vegetation index (NDVI) is computed using spectral reflectance in red and near infrared wavelengths. 

The NDVI equation is:

$$NDVI=\frac{NIR-red}{NIR+red}$$

NDVI values have a range of -1 to 1; a higher NDVI value indicates greater vegetation cover, greenness, or biomass within a pixel.

The NDVI is based on the reflectance characteristics of green vegetation which absorbs red and reflects near infrared light. Red light is absorbed by chlorophyll in leaves (which also explains why we see vegetation in green). Near infrared electromagnetic radiation is scattered by mesophyll tissue and openings between cells. Some of this scattered near infrared radiation is reflected upwards and detected by sensors. For vegetated surfaces there will be a larger difference between red and near infrared reflectance than over non-vegetated surfaces. 

The following two functions compute the NDVI for Landsat 7 and Landsat 8. They return an `Image` with a band named `'nd'` as set by the `rename()` function and a `system:time_start` property which is set in the returned `Image`s metadata. This is important as it records what date and time the NDVI data corresponds to.

Note the different band designations for Landsat 7 and Landsat 8. The near infrared band in Landsat 7 is band 4 (`'B4'`) and the red band is band 3 (`'B3'`). For Landsat 8 the near infrared band is band 5 (`'B5'`) and the red band in band 4 (`'B4'`).



```js
// Image math - NDVI
var ndviL7 = function(image) {
  
  var ndvi = image.select('SR_B4').subtract(image.select('SR_B3'))
    .divide(image.select('SR_B4').add(image.select('SR_B3')));
  ndvi = ndvi.rename('nd');
  var startDate = image.get('system:time_start'); 
  return ndvi.set({'system:time_start': startDate});
};

var ndviL8 = function(image) {
  
  var ndvi = image.select('SR_B5').subtract(image.select('SR_B4'))
    .divide(image.select('SR_B5').add(image.select('SR_B4')));
  ndvi = ndvi.rename('nd');
  var startDate = image.get('system:time_start'); 
  return ndvi.set({'system:time_start': startDate});
};

```

Map these functions over the Landsat 7 and Landsat 8 `ImageCollection`s to return an `ImageCollection` of NDVI `Image`s. Display the first NDVI `Image` returned in the `l82017NDVI` on the map to visualise the output of computing the NDVI using Landsat data.



```js
//Map NDVI functions of Landsat ImageCollections
var l2000 = l72000.map(ndviL7);

var l72017NDVI = l72017.map(ndviL7);

var l82017NDVI = l82017.map(ndviL8);
print(l82017NDVI);

// display first Landsat 8 NDVI Image on the map
Map.addLayer(l82017NDVI.first(), {min: 0.2, max: 0.8, palette:['#f7fcfd','#e5f5f9','#ccece6','#99d8c9','#66c2a4','#41ae76','#238b45','#006d2c','#00441b']}, 'first L8 NDVI Image');

```
 
## Join / Combine

The time period for Landsat 7 observations spans 1999 to 2024. You have loaded Landsat 7 observations for the period 2000-2002 and 2017-2019. Landsat 8 observations start from 2013. You need to combine the Landsat 7 and Landsat 8 observations for period 2017 to 2019. To do this use the `merge()` function which merges two `ImageCollection`s into one. You can then sort by the date of `Image` capture so the `Image`s in the returned collection are in a temporal order.

```js
// merge Landsat 7 NDVI and Landsat 8 NDVI ImageCollections and sort by time
var l2017 = ee.ImageCollection(l82017NDVI.merge(l72017NDVI)).sort('system:time_start');
print(l2017);

```

Inspect the `ImageCollection` `l2017` in the *console* to see that it includes Landsat 8 and Landsat 7 `Image`s. 

## Summarise 

You now have two `ImageCollection`s `l2000` and `l2017` that contain NDVI `Image`s from 2000 to 2002 and 2017 to 2019, respectively. You need to summarise the NDVI data in these two `ImageCollection`s to create a per-pixel measure of vegetation condition in each time-period. 

The process of combining multiple, spatially overlapping pixel measures is called <a href="https://developers.google.com/earth-engine/guides/ic_composite_mosaic" target="_blank">compositing</a>. Composite vegetation index `Image`s are computed because spectral reflectance values for a signal time point are often noisy (e.g. due to cloud cover or atmospheric contamination). However, the summary of multiple measures at the same location reduces noise and provides a more accurate indication of pixel characteristics. 

Creating composite `Image`s from `ImageCollection`s in Google Earth Engine is straightforward. You can apply the `median()`, `mean()`, `max()` functions to an `ImageCollection` to compute the per-pixel median, mean, or max value for all `Image`s in the collection. Here, use the `median()` function to create a median NDVI composite for the period 2000 to 2002 and 2017 to 2019. 

```js
// 3 year median NDVI composite
var l2000Composite = l2000
  .median()
  .clip(bBox);
  
// mask water
l2000Composite = l2000Composite.updateMask(l2000Composite.gt(0));

```

You will spot that you also `clip`ped your median NDVI composite `Image` using the extent of the `Geometry` object `bBox`. You also masked out any pixels in your median NDVI composite with NDVI value less than or equal to zero. This is a quick way to remove water from your `Image` as water's NDVI values are typically below 0. Both of these steps are to enhance visualisation of your NDVI composite. 

The `clip()` operation is another example of spatial subsetting. 

Creating the mask of pixel locations with a median NDVI composite value greater than 0 `l2000Composite.gt(0)` is an example of a local raster operation using a comparison operator as opposed to an arithmetic operator (each pixel value will evaluate to true or false).

Turn off the other layers on your map display using the *Layers* menu. Visualise your median NDVI composite `Image` on the map. Use the inspector to query NDVI values at different locations.


```js
// 3 year median NDVI composite - 2000 - 2002
print(l2000Composite);
Map.centerObject(l2000Composite, 12);
Map.addLayer(l2000Composite, {min: 0.1, max: 0.8, palette:['#ffffcc','#d9f0a3','#addd8e','#78c679','#41ab5d','#238443','#005a32']}, 'median NDVI 2000 - 2002');
```

Repeat the steps for the time period 2017 to 2019. 

```js
// 3 year median NDVI composite - 2017 - 2019
var l2017Composite = l2017
  .median()
  .clip(bBox);
  
// mask water
l2017Composite = l2017Composite.updateMask(l2017Composite.gt(0));  

print(l2017Composite);
Map.addLayer(l2017Composite, {min: 0.1, max: 0.8, palette:['#ffffcc','#d9f0a3','#addd8e','#78c679','#41ab5d','#238443','#005a32']}, 'median NDVI 2017 - 2019');
```

Let the median NDVI composites for the period 2000-2002 and 2017-2019 (`l2000Composite` and `l2017Composite`) load on your map display. Toggle them on and off using the *Layers* menu to visualise change in vegetation over that time period. You should see something similar to the video below. Where you see change in NDVI can you explain why? Look at the satellite base map in areas where you note change in NDVI to see if that can provide any clues as to what caused the change in NDVI. 


<center>
<iframe src="https://player.vimeo.com/video/454224096" width="640" height="423" frameborder="0" allow="autoplay; fullscreen" allowfullscreen></iframe>
<p>Change in NDVI between 2000-2002 and 2017-2019</p>
</center>


## Change Detection

You can use the median NDVI composite `Image`s for the two time periods to answer the question at the beginning of the lab: <b>*Where has vegetation cover changed between 2000-2002 and 2017-2019?*</b>. 

You can compute a <a href="https://drive.google.com/file/d/1mL8yc-y1Tk45ofDpN8aGFBiaqTaC9nbw/view?usp=drive_link" target="_blank">change detection</a> `Image` which shows the location, direction (positve NDVI change, negative NDVI change, no change), and magnitude of NDVI change between these two time periods. Bi-temporal change detection is the operation of detecting change between two `Image`s captured on different dates. 

You can perform an `Image` differencing operation to compute a change `Image`.

```js
// change detection
var ndviChange = l2017Composite.subtract(l2000Composite);
print(ndviChange);
Map.addLayer(ndviChange, {min: -0.2, max: 0.2, palette:['#d73027','#f46d43','#fdae61','#fee090','#ffffbf','#e0f3f8','#abd9e9','#74add1','#4575b4']}, 'change in NDVI');

```

You should see an `Image` similar to the figure below on your map visualising change in NDVI between 2000-2002 and 2017-2019. The areas in red indicate a decrease in NDVI, yellow little / no change in NDVI, and blue indicates an increase in NDVI. 

Change the min and max values in the visualisation parameters to highlight different features of vegetation change (i.e. just areas of vegetation loss).

<figure markdown="span">
  ![Image difference between NDVI in 2000-2002 and 2017-2019. Red indicates decrease in NDVI and blue indicates increase in NDVI.](../images/ndvi-difference-img.png)
  <figcaption><code>Image</code> difference between NDVI in 2000-2002 and 2017-2019. Red indicates decrease in NDVI and blue indicates increase in NDVI.</figcaption>
</figure>