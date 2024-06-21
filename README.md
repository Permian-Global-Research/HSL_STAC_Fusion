# HSL_STAC_Fusion

<img align="Right"  src= "https://github.com/Permian-Global-Research/HSL_STAC_Fusion/assets/69790440/29ef2293-6a01-4080-8ab1-a8a737a35165" width="250" height="250" />

Script to Fuse the 2 branches (HSL.S &amp; HSL.L) of the NASA -Harmonized Sentinel Landsat (HSL) dataset into a single datacube, and generate Median composites that include spectral indexes such as NDVI, EVI.





> [!IMPORTANT]
> Key information users need to know to achieve their goal.

> [!CAUTION]
> Advises about risks or negative outcomes of certain actions.

## Aim

## Section 

* Harmonised Sentinel and Landsat Catalogue Fusion
    * _Bands selection and re-naming_
* Cloud masking
* Temporal Averaging | Median composite of harmonised Sentinel-Landsat 
* Total imagery coverage | Region of Interest
* Spectral indexes | Normalised Difference Vegetation Index (NDVI) & Enhanced Vegetation Index (EVI)

#### Inputs - How to tweak this project for your own uses


## Example
This is a basic example which shows you how to generate your HSL-Fused composites

##### Libraries and Packages
```python
import odc.geo  # noqa
from odc.algo import mask_cleanup
from odc.stac import configure_rio, load
from pystac_client import Client
import datetime
import os
import rioxarray as rxr
import matplotlib.pyplot as plt
import xarray as xr
import time
import requests
from requests.exceptions import HTTPError
from tqdm import tqdm
import logging
import geopandas as gpd
import csv
```
#####  NASA's EarthData Token Authorization
> [!NOTE]
> Generating a User token from EDL GUI
Visit Earthdata Login https://urs.earthdata.nasa.gov and Login with your user credentials.
> User tokens are valid for a duration of 60 days.

```python
    # Insert NASA-ASFCatalogue Token 
import os
os.environ['EARTHDATA_TOKEN'] = #'Insert your key'

```

##### Access NASA's Harmonised Sentinel-Landsat data catalogue
```python
    # Search collections

catalog = "https://cmr.earthdata.nasa.gov/cloudstac/LPCLOUD/"

# Searching across both landsat and sentinel at 30 m
collections = ["HLSS30.v2.0", "HLSL30.v2.0"]

client = Client.open(catalog)
collections
```

##### Search parameters | [ROI name, Year, Dates ]

```python 

  # Specify MR year
year = #'year' (eg. 2024)

# Specify name of Region of Interest 

roi = #'ROI name' (eg. Project)

  # Select dates

# Construct date string for datetime range
start_date = "2024-01-01"
#end_date = "YYYY-MM-DD"
end_date = datetime.date.today().strftime("%Y-%m-%d")

date_string = f"{start_date}/{end_date}"
```

#### Region of Interest coordinates

> [!TIP]
> There are multiple options for extracting your ROI bounding box. Quick solution -->
> http://bboxfinder.com/

```python
  # Bounding Box coordinates
ll = #(northing, easting)(e.g (4.932133, 117.185748))
ur = #(norting, easting) (e.g.(5.311166, 117.632919))

bbox = [ll[1], ll[0], ur[1], ur[0]]


# Search for items in the collection
items = client.search(
    collections=collections, 
    bbox=bbox, 
    datetime=date_string).items() 

 # Print search
    
items = [i for i in items]
print(f"Found {len(items)} items")
print(items)
for i in items:
  print(i.get_collection())
```

#### Extract oindicental bands from Sentinel-2 and Landsat missions

```python
   # Filtering bands

s30_bands = ['B12', 'B11', 'B8A', 'B04', 'B03','B02', 'B01', 'Fmask']    # S30 bands for EVI calculation and quality filtering -> SWIR2, SWIR 1, NIR, RED, GREEN, BLUE, Coastal, Quality 
l30_bands = ['B07', 'B06', 'B05', 'B04', 'B03','B02', 'B01', 'Fmask']    # L30 bands for EVI calculation and quality filtering -> SWIR2, SWIR 1, NIR, RED, GREEN, BLUE, Coastal, Quality

   # Print & Test
l30_bands 
```

#### Filter HSL collections by new bands
```
# And now to loop through and filter the items collection by bands:
new_band_links = []

for i in items:
        if i.collection_id == 'HLSS30.v2.0':
            #print(i.properties['eo:cloud_cover'])
            new_bands = s30_bands
        elif i.collection_id == 'HLSL30.v2.0':
            #print(i.properties['eo:cloud_cover'])
            new_bands = l30_bands

        for a in i.assets:
            if any(b==a for b in new_bands):
                new_band_links.append(i.assets[a].href)
```
#### Load HSL collections and apply Cloud-Mask
> [!WARNING]
> Please pay attention to the coordinate reference system (CRS) used. Make sure it aligns with your ROI-Shapefile. 
```
   # Configure GDAL. You need to export your earthdata token as an environment variable.
header_string = f"Authorization: Bearer {os.environ['EARTHDATA_TOKEN']}"
configure_rio(cloud_defaults=True, GDAL_HTTP_HEADERS=header_string)

data = load(
    items,
    bbox=bbox,
    crs="epsg:32650",
    resolution=30,
    chunks={"x": 2500, "y": 2500, "time": 1},
    groupby="solar_day",
    bands=["SWIR2", "SWIR 1", "NIR", "RED", "GREEN", "BLUE", "Coastal", "Fmask"],
)

   # Get cloud  mask bitfields
mask_bitfields = [0, 1,  3]
bitmask = 0
for field in mask_bitfields:
    bitmask |= 1 << field

# Get cloud mask
cloud_mask = data["Fmask"].astype(int) & bitmask != 0

# Contract and then expand the cloud mask to remove small areas
dilated = mask_cleanup(cloud_mask, [("opening", 2), ("dilation", 3)])

masked = data.where(~dilated)
masked
```
