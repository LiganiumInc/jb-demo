---
jupyter:
  jupytext:
    formats: ipynb,md
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.15.2
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
---

[![image](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/gee-community/geemap/blob/master/examples/workshops/Crop_Mapping_2022.ipynb)
[![image](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/gee-community/geemap/master?urlpath=lab%2Ftree%2Fexamples%2Fworkshops)


# Cropland mapping with Google Earth Engine and geemap

```{contents}
```

Useful links

This notebook is a collection of ..........

- [Google Earth Engine](https://earthengine.google.com/)
- [Sign up for an Earth Engine account](https://earthengine.google.com/signup/)
- [Awesome GEE Community Datasets](https://samapriya.github.io/awesome-gee-community-datasets)
- [Geemap website](https://geemap.org/)
- [Geemap book](https://book.geemap.org/)
- [Geemap YouTube videos](https://youtube.com/@giswqs)

```python
# !pip install geemap
```

## Import libraries

```python
import ee
import geemap
```

```python
import os
```

Test something new


### blabla


## ESA WordCover

The European Space Agency (ESA) WorldCover 10 m 2020 product provides a global land cover map for 2020 at 10 m resolution based on Sentinel-1 and Sentinel-2 data. The WorldCover product comes with 11 land cover classes and has been generated in the framework of the ESA WorldCover project, part of the 5th Earth Observation Envelope Programme (EOEP-5) of the European Space Agency.

- [ESA WorldCover website](https://esa-worldcover.org/)
- [EAS WroldCover in the Earth Engine Data Catalog](https://developers.google.com/earth-engine/datasets/catalog/ESA_WorldCover_v100)
- [User Manual and Validation Report](https://esa-worldcover.org/en/data-access)


### Using Web Map Services

The ESA WorldCover product can also be used within other websites or GIS clients by 'Web Map Services'. These services provide a direct link to the cached images and are the best option if you simply want to map the data and produce cartographic products. They are not suitable for analysis as the data are represented only as RGB images. 

- WMTS: https://services.terrascope.be/wmts/v2
- WMS: https://services.terrascope.be/wms/v2
- Layers: WORLDCOVER_2020_MAP, WORLDCOVER_2020_S2_FCC, WORLDCOVER_2020_S2_TCC

```python
Map = geemap.Map()

esa_wms = 'https://services.terrascope.be/wms/v2'  # The WMS URL
tcc_layer = 'WORLDCOVER_2020_S2_TCC'  # The true color composite imagery
fcc_layer = 'WORLDCOVER_2020_S2_FCC'  # The false color composite imagery
map_layer = 'WORLDCOVER_2020_MAP'  # The land cover classification map

Map.add_wms_layer(esa_wms, layers=tcc_layer, name='True Color', attribution='ESA')
Map.add_wms_layer(esa_wms, layers=fcc_layer, name='False Color', attribution='ESA')
Map.add_wms_layer(esa_wms, layers=map_layer, name='Classification', attribution='ESA')

Map.add_legend(title='ESA Land Cover', builtin_legend='ESA_WorldCover')
Map
```

### Using Earth Engine

- [EAS WroldCover in the Earth Engine Data Catalog](https://developers.google.com/earth-engine/datasets/catalog/ESA_WorldCover_v100)

```python
Map = geemap.Map()
Map.add_basemap('HYBRID')

esa = ee.ImageCollection("ESA/WorldCover/v100").first()
esa_vis = {'bands': ['Map']}

Map.addLayer(esa, esa_vis, "ESA Land Cover")
Map.add_legend(title="ESA Land Cover", builtin_legend='ESA_WorldCover')

Map
```

### Creating charts

```python
histogram = geemap.image_histogram(
    esa, scale=1000, x_label='Land Cover Type', y_label='Area (km2)'
)
histogram
```

```python
df = geemap.image_histogram(esa, scale=1000, return_df=True)
df
```

```python
esa_labels = list(geemap.builtin_legends['ESA_WorldCover'].keys())
esa_labels
```

```python
df['label'] = esa_labels
df
```

```python
round(df['value'].sum() / 1e6, 2)
```

```python
geemap.bar_chart(
    df, x='label', y='value', x_label='Land Cover Type', y_label='Area (km2)'
)
```

```python
geemap.pie_chart(df, names='label', values='value', height=500)
```

### Adding Administrative Boundaries

```python
countries = ee.FeatureCollection(geemap.examples.get_ee_path('countries'))
africa = countries.filter(ee.Filter.eq('CONTINENT', 'Africa'))
style = {'fillColor': '00000000'}
Map.addLayer(countries.style(**style), {}, 'Countries', False)
Map.addLayer(africa.style(**style), {}, 'Africa')
Map.centerObject(africa)
Map
```

### Extracting Croplands

```python
cropland = esa.eq(40).clipToCollection(africa).selfMask()
Map.addLayer(cropland, {'palette': ['f096ff']}, 'Cropland')
Map.show_layer(name='ESA Land Cover', show=False)
```

### Zonal Statistics

```python
geemap.zonal_stats(
    cropland, africa, 'esa_cropland.csv', statistics_type='SUM', scale=1000
)
```

```python
df = geemap.csv_to_df('esa_cropland.csv')
df.head()
```

```python
geemap.bar_chart(
    df, x='NAME', y='sum', max_rows=30, x_label='Country', y_label='Area (km2)'
)
```

```python
geemap.pie_chart(df, names='NAME', values='sum', max_rows=20, height=500)
```

## ESRI GLobal Land Cover

The ESRI GLobal Land Cover dataset is a global map of land use/land cover (LULC) derived from ESA Sentinel-2 imagery at 10m resolution. Each year is generated from Impact Observatory’s deep learning AI land classification model used a massive training dataset of billions of human-labeled image pixels developed by the National Geographic Society. The global maps were produced by applying this model to the Sentinel-2 scene collection on Microsoft’s Planetary Computer, processing over 400,000 Earth observations per year. 

- https://livingatlas.arcgis.com/landcover/
- https://www.arcgis.com/home/item.html?id=d3da5dd386d140cf93fc9ecbf8da5e31
- https://samapriya.github.io/awesome-gee-community-datasets/projects/S2TSLULC/


### Using Awesome GEE Community Datasets

```python
Map = geemap.Map()
Map.add_basemap('HYBRID')

esri = ee.ImageCollection(
    'projects/sat-io/open-datasets/landcover/ESRI_Global-LULC_10m_TS'
)

esri_2017 = esri.filterDate('2017-01-01', '2017-12-31').mosaic()
esri_2018 = esri.filterDate('2018-01-01', '2018-12-31').mosaic()
esri_2019 = esri.filterDate('2019-01-01', '2019-12-31').mosaic()
esri_2020 = esri.filterDate('2020-01-01', '2020-12-31').mosaic()
esri_2021 = esri.filterDate('2021-01-01', '2021-12-31').mosaic()

esri_vis = {'min': 1, 'max': 11, 'palette': 'esri_lulc'}

Map.addLayer(esri_2017, esri_vis, 'ESRI LULC 2017')
Map.addLayer(esri_2018, esri_vis, 'ESRI LULC 2018')
Map.addLayer(esri_2019, esri_vis, 'ESRI LULC 2019')
Map.addLayer(esri_2020, esri_vis, 'ESRI LULC 2020')
Map.addLayer(esri_2021, esri_vis, 'ESRI LULC 2021')

Map.add_legend(title='ESRI Land Cover', builtin_legend='ESRI_LandCover_TS')
Map
```

### Using Timeseries Inspector

```python
images = ee.List([esri_2017, esri_2018, esri_2019, esri_2020, esri_2021])
collection = ee.ImageCollection.fromImages(images)
```

```python
years = [str(year) for year in range(2017, 2022)]
years
```

```python
Map = geemap.Map()
Map.ts_inspector(collection, years, esri_vis, width='80px')
Map.add_legend(title='ESRI Land Cover', builtin_legend='ESRI_LandCover_TS')
Map
```

### Extracting Croplands

```python
countries = ee.FeatureCollection(geemap.examples.get_ee_path('countries'))
africa = countries.filter(ee.Filter.eq('CONTINENT', 'Africa'))
```

```python
cropland_col = collection.map(lambda img: img.eq(5).clipToCollection(africa).selfMask())
cropland_ts = cropland_col.toBands().rename(years)
```

```python
Map = geemap.Map()

style = {'fillColor': '00000000'}
Map.addLayer(cropland_col.first(), {'palette': ['#ab6c28']}, 'first')
Map.addLayer(countries.style(**style), {}, 'Countries', False)
Map.addLayer(africa.style(**style), {}, 'Africa')
Map.centerObject(africa)

Map
```

```python
cropland_ts.bandNames().getInfo()
```

### Zonal Statistics

```python
geemap.zonal_stats(
    cropland_ts, africa, 'esri_cropland.csv', statistics_type='SUM', scale=1000
)
```

```python
df = geemap.csv_to_df('esri_cropland.csv')
df.head()
```

```python
geemap.bar_chart(df, x='NAME', y=years, max_rows=20, legend_title='Year')
```

```python
geemap.pie_chart(df, names='NAME', values='2020', max_rows=20, height=500)
```

### Analyzing Cropland Gain and Loss

```python
Map = geemap.Map()
Map.add_basemap('HYBRID')

cropland_2017 = esri_2017.eq(5).selfMask()
cropland_2021 = esri_2021.eq(5).selfMask()

cropland_gain = esri_2017.neq(5).And(esri_2021.eq(5)).selfMask()
cropland_loss = esri_2017.eq(5).And(esri_2021.neq(5)).selfMask()

Map.addLayer(cropland_2017, {'palette': 'brown'}, 'Cropland 2017', False)
Map.addLayer(cropland_2021, {'palette': 'cyan'}, 'Cropland 2021', False)

Map.addLayer(cropland_gain, {'palette': 'yellow'}, 'Cropland gain')
Map.addLayer(cropland_loss, {'palette': 'red'}, 'Cropland loss')
Map
```

```python
geemap.zonal_stats(
    cropland_gain,
    countries,
    'esri_cropland_gain.csv',
    statistics_type='SUM',
    scale=1000,
)
```

```python
df = geemap.csv_to_df('esri_cropland_gain.csv')
df.head()
```

```python
geemap.bar_chart(
    df,
    x='NAME',
    y='sum',
    max_rows=30,
    x_label='Country',
    y_label='Area (km2)',
    title='Cropland Gain',
)
```

```python
geemap.pie_chart(
    df, names='NAME', values='sum', max_rows=30, height=500, title='Cropland Gain'
)
```

```python
geemap.zonal_stats(
    cropland_loss,
    countries,
    'esri_cropland_loss.csv',
    statistics_type='SUM',
    scale=1000,
)
```

```python
df = geemap.csv_to_df('esri_cropland_loss.csv')
df.head()
```

```python
geemap.bar_chart(
    df,
    x='NAME',
    y='sum',
    max_rows=30,
    x_label='Country',
    y_label='Area (km2)',
    title='Cropland Loss',
)
```

```python
geemap.pie_chart(
    df, names='NAME', values='sum', max_rows=30, height=500, title='Cropland Loss'
)
```

## Dynamic World Land Cover

Dynamic World is a near realtime 10m resolution global land use land cover dataset, produced using deep learning, freely available and openly licensed. As a result of leveraging a novel deep learning approach, based on Sentinel-2 Top of Atmosphere, Dynamic World offers global land cover updating every 2-5 days depending on location. 

- [Dynamic World Website](https://www.dynamicworld.app/)
- [Dynamic World datasets on Earth Engine](https://developers.google.com/earth-engine/datasets/catalog/GOOGLE_DYNAMICWORLD_V1)


### Classification and Probability

```python
Map = geemap.Map()

region = ee.Geometry.BBox(-179, -89, 179, 89)
start_date = '2021-01-01'
end_date = '2022-01-01'

dw_class = geemap.dynamic_world(region, start_date, end_date, return_type='class')
dw = geemap.dynamic_world(region, start_date, end_date, return_type='hillshade')

dw_vis = {"min": 0, "max": 8, "palette": 'dw'}

Map.addLayer(dw_class, dw_vis, 'DW Land Cover', False)
Map.addLayer(dw, {}, 'DW Land Cover Hillshade')

Map.add_legend(title="Dynamic World Land Cover", builtin_legend='Dynamic_World')
Map.setCenter(-88.9088, 43.0006, 12)
Map
```

### ESA Land Cover vs. Dynamic World

```python
Map = geemap.Map(center=[39.3322, -106.7349], zoom=10)

left_layer = geemap.ee_tile_layer(esa, esa_vis, "ESA Land Cover")
right_layer = geemap.ee_tile_layer(dw, {}, "Dynamic World Land Cover")

Map.split_map(left_layer, right_layer)
Map.add_legend(
    title="ESA Land Cover", builtin_legend='ESA_WorldCover', position='bottomleft'
)
Map.add_legend(
    title="Dynamic World Land Cover",
    builtin_legend='Dynamic_World',
    position='bottomright',
)
Map.setCenter(-88.9088, 43.0006, 12)

Map
```

### ESRI Land Cover vs. Dynamic World

```python
Map = geemap.Map(center=[-89.3998, 43.0886], zoom=10)

left_layer = geemap.ee_tile_layer(esri_2021, esri_vis, "ESRI Land Cover")
right_layer = geemap.ee_tile_layer(dw, {}, "Dynamic World Land Cover")

Map.split_map(left_layer, right_layer)
Map.add_legend(
    title="ESRI Land Cover", builtin_legend='ESRI_LandCover', position='bottomleft'
)
Map.add_legend(
    title="Dynamic World Land Cover",
    builtin_legend='Dynamic_World',
    position='bottomright',
)
Map.setCenter(-88.9088, 43.0006, 12)

Map
```
