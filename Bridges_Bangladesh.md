---
jupyter:
  jupytext:
    formats: ipynb,md
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.2'
      jupytext_version: 1.3.3+dev
  kernelspec:
    display_name: Python 3
    language: python
    name: python3
---

Fixed Errors - 
1. Exchanged latitudes and longotudes
2. Replaced Nan latitudes and longitudes with zeroes

```python
import os
import pandas as pd
import geopandas as gpd
import matplotlib.pyplot as plt
import numpy as np
from geopy.distance import vincenty
```

```python
data_path = os.getcwd()
```

Potential Errors -
1. Duplicate latitude and longitude 
2. Zero latitude and longitude 


Possible approaches - 
1. Check the difference in latitude and longitude and it must be meaningful 
2. Check the latitude difference with the chainage of the bridge part that is calculated. 
3. No IDs for some of the bridges (Column: name)
4. Is it true that one road has one name?


```python
bridges = pd.read_excel(data_path + '\\DataSource\\BMMS_overview_SortedRoad.xlsx')
```

```python
for i in range(len(bridges)):
    if bridges['lat'][i]>90 :
        c= bridges['lat'][i] 
        bridges['lat'][i] = bridges['lon'][i] 
        bridges['lon'][i] =c
        
```

```python
bridges['lat'] = bridges['lat'].fillna(0)
bridges['lon'] = bridges['lon'].fillna(0)
```

```python
def distance_calc (lat1, lon1, lat2, lon2):
    start = (lat1, lon1)
    stop = (lat2, lon2)

    return vincenty(start, stop).kilometers
```

```python
lon = pd.concat([bridges['lon'].shift(), bridges.loc[1:, 'lon']], axis=1, ignore_index=True)
lat = pd.concat([bridges['lat'].shift(), bridges.loc[1:, 'lat']], axis=1, ignore_index=True)
```

```python
lat.columns = ['startlat', 'stoplat']
lon.columns = ['startlon', 'stoplon']
```

```python
lon.drop(lon.index[0],inplace= True)
lat.drop(lat.index[0],inplace= True)
```

```python
lon = lon.reset_index(drop = True)
lat = lat.reset_index(drop = True)
```

```python
lat.head()
```

```python
lon.head()
```

```python
bridges['dist']= np.zeros
```

```python
for i in range(len(lon)):
        bridges['dist'][i] = distance_calc(lat['startlat'][i], lon['startlon'][i],lat['stoplat'][i], lon['stoplon'][i] )
```

```python
print("Hello")
```
