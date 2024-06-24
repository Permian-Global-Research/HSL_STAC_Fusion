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

#### Extract coindicental bands from Sentinel-2 and Landsat missions

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

#### Create Annual Median Composite

```python

# Enable logging to track progress
logging.basicConfig(format='%(asctime)s - %(message)s', level=logging.INFO)

def retry_with_backoff(url, max_retries=5):
    """Retry a request with exponential backoff."""
    for i in range(max_retries):
        try:
            response = requests.get(url)
            response.raise_for_status()
            return response
        except HTTPError as e:
            logging.warning(f"HTTP error ({e.response.status_code}) - {url}. Retrying again in {2**i} secs")
            time.sleep(2**i)
    logging.error(f"Failed to retrieve data from {url} after {max_retries} retries")
    return None

# Create a simple cloud-free median now we have masked data
logging.info("Computing cloud-free median...")
num_steps = masked.time.size
with tqdm(total=num_steps) as pbar:
    median = []
    for step in range(num_steps):
        try:
            median.append(masked.isel(time=step))
            pbar.update(1)
        except Exception as e:
            logging.error(f"Error processing step {step}: {e}")
median = xr.concat(median, dim='time').median("time").compute()

```

![image](https://github.com/Permian-Global-Research/HSL_STAC_Fusion/assets/69790440/5b7b12cc-5bdb-4552-b84b-025691bbff16)



#### Calculate the total ROI area covered

```
    # Show Image and Total ROI area covered by image composite 

roi = 'Kuamut'
# Calculate the area of the 'Kuamut project area' polygon
project_area_sqm = gdf.geometry.to_crs('EPSG:32650').area.sum()

# Calculate the area covered by the satellite image within the 'Kuamut project area'
clipped_image = median.rio.clip(gdf.geometry)
resolution = 30  # Example resolution, replace with actual resolution if known
image_area_sqm = (clipped_image["RED"].count() * resolution ** 2)

# Calculate the percentage of the area covered by the satellite image relative to the total area of the 'Kuamut project area'
percentage_covered = (image_area_sqm / project_area_sqm) * 100

# Plot the satellite image
fig, ax = plt.subplots(figsize=(10, 10))
vmin, vmax = median["RED"].min(), median["RED"].max()
rgb = median[["RED", "GREEN", "BLUE"]].to_array()
rgb.plot.imshow(ax=ax, vmin=0, vmax=1000, extent=(median.x.min(), median.x.max(), median.y.min(), median.y.max()))
gdf.plot(ax=ax, color='none', edgecolor='red')

# Annotate the plot with the calculated percentage of coverage
plt.text(0.95, 0.95, f'Total Coverage: {percentage_covered:.2f}%', horizontalalignment='right', verticalalignment='top', transform=ax.transAxes, fontsize=12, bbox=dict(facecolor='white', alpha=0.8))

# Set the plot title
plt.title(f'HSL Cloud-Free Composite {roi} {year}')

# Rename the axis labels
plt.xlabel('X coordinates')
plt.ylabel('Y coordinates')

# Save the plot as a high-resolution JPEG image
output_path = f"HSL_CloudFreeComposite_{roi}{year}.jpg"
plt.savefig(output_path, dpi=300)  # Set the dpi parameter to adjust the resolution

plt.show()
```
![image](https://github.com/Permian-Global-Research/HSL_STAC_Fusion/assets/69790440/474890ee-4ce6-437f-942c-47d65b44fd73)



#### NDVI calculation

```
    # Normalised Difference Vegetation Index (NDVI)

# Select bands B05 (NIR) and B04 (Red) from the median image
nir_median = median['NIR']
red_median = median['RED']

# Calculate NDVI for the median image
ndvi_median = (nir_median - red_median) / (nir_median + red_median)

# Add NDVI as a new variable to the median image dataset
median_with_ndvi = median.assign(NDVI=ndvi_median)

# Set up a larger figure size
plt.figure(figsize=(10, 8))

# Visualize the NDVI band with scale adjusted to 0 to 1
plt.imshow(median_with_ndvi.NDVI, cmap='RdYlGn', extent=(0, median_with_ndvi.NDVI.shape[1], 0, median_with_ndvi.NDVI.shape[0]), vmin=0, vmax=1)
plt.colorbar(label='NDVI')
plt.title(f'NDVI of Median Composite {roi}{year}')
plt.show()
```
![image](https://github.com/Permian-Global-Research/HSL_STAC_Fusion/assets/69790440/01de48fb-c636-4d07-b93d-0a7dee109811)

#### EVI calculation
```
    # Enhanced Vegetation Index (EVI)
    
# Constants for EVI calculation
G = 2.5
C1 = 6
C2 = 7.5
L = 1

import matplotlib.pyplot as plt

# Select bands B05 (NIR) and B04 (Red) from the median image
nir = median_with_ndvi['NIR']
red = median_with_ndvi['RED']
blue = median_with_ndvi['BLUE']

    # Calculate EVI
evi_median = G * ((nir - red) / (nir + C1 * red - C2 * blue + L))

# Add NDVI as a new variable to the median image dataset
median_with_evi = median_with_ndvi.assign(EVI=evi_median)

# Set up a larger figure size
plt.figure(figsize=(10, 8))

# Visualize the NDVI band with scale adjusted to 0 to 1
plt.imshow(median_with_evi.EVI, cmap='RdYlGn', extent=(0, median_with_evi.EVI.shape[1], 0, median_with_evi.EVI.shape[0]), vmin=0, vmax=3)
plt.colorbar(label='EVI')
plt.title(f'EVI of Median Composite {roi}{year}')
plt.show()
```
![image](https://github.com/Permian-Global-Research/HSL_STAC_Fusion/assets/69790440/145eea02-1c70-481e-9ed2-d1134afee09d)


#### Save your composite as a Cloud-optimised GeoTiff in your repository
```
    # Save Composite image with both NDVI and EVI indexes

import os
import rioxarray as rxr
# Assuming median_with_ndvi contains the raster dataset

# Get the current working directory
current_dir = os.getcwd()

# Define the file path for the COG
cog_path = os.path.join(current_dir, f"{roi}_HSL_Median{year}_V6.tif")

# Save the dataset as a Cloud Optimized GeoTIFF
median_with_evi.rio.to_raster(cog_path, driver="GTiff", tif_cog_profile="deflate")

```

## Find a bug?
If you found an issue or would like to submit an improvement to this project, please submit an issue using the issues tab above. If you would like to submit a PR with a fix, reference the issue you created! Thanks!
