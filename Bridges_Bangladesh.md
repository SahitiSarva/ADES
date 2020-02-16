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

### Import Libraries

```python
import os
import pandas as pd
import geopandas as gpd
import numpy as np
from geopy.distance import geodesic
import matplotlib.pyplot as plt
from tqdm.auto import tqdm
from geographiclib.geodesic import Geodesic
import plotly.express as px
```

### Data Loading
**Establish current file directory and read-in data file provided. Enter bridges file name as "data_file"**

#### Update: Data loading and fudging complete for now. Skip to Data Correcting section and read in "bridges_cleaned.xlsx" (below) instead

```python
data_path = os.getcwd() + "\\DataSource\\"
```

```python
data_file = "BMMS_overview.xlsx"
```

```python
bridges = pd.read_excel(data_path + data_file)
```

#### Data Exploration
**Which columns are the unique identifiers?**

```python
bridges.nunique()
```

**"structureNr" is the unique identifier for each line item.**


### Data Cleaning
**Some bridges had latitude and longitudes flipped. This is addressed as follows:**

```python
for i in range(len(bridges)):
    if bridges['lat'][i]>90:
        c = bridges['lat'][i] 
        bridges['lat'][i] = bridges['lon'][i] 
        bridges['lon'][i] = c
```

**Remove rows with no coordinates or with 0,0 coordinates.**

```python
bridges = bridges[np.isfinite(bridges['lat'])]
```

```python
bridges = bridges[bridges["lat"]!=0]
```

```python
bridges = bridges.sort_values(["road","km"]).reset_index(drop=True)
```

### Data Fudging

**Thought Process**:
1. The BMMS_overview dataset has multiple line items with the same ID ("road" column), which we assume to be a single connected bridge.
2. It appears that the length of each bridge line item is approximately represented by the distance between its lat-long coordinates and that of the subsequent item.
3. We observed that the length of a bridge is approximately equal to the preceding line item's length + the distance between that bridge and the line item that precedes it. Thus, we calculate the distances between each line item and its preceding line item. If the numbers do not approximately add up, we assume there is an error in the coordinates.

```python
# bridges["coordinates"] = np.zeros
# bridges["coordinates"] = list(zip(bridges["lat"], bridges["lon"]))
```

```python
bridges_self = pd.concat([bridges, bridges.shift(-1).add_suffix('2')],
                         axis=1)[["lat", "lon", "lat2", "lon2", "road", "road2", "structureNr"]]
```

```python
bridges_self = bridges_self[bridges_self.road==bridges_self.road2]
```

```python
def distance_calc (row):
    start = (row['lat'], row['lon'])
    stop = (row['lat2'], row['lon2'])
    return geodesic(start, stop).kilometers
```

```python
bridges_self['dist'] = bridges_self.apply (lambda row: distance_calc (row),axis=1)
```

```python
bridges = pd.merge(bridges, bridges_self[["dist", "structureNr"]], on = "structureNr", how = "left")
```

```python
bridges["verification"] = bridges["km"] + bridges["dist"]
bridges["verification"] = bridges["verification"].shift(1)
```

```python
bridges[["road", "km", "dist", "verification"]].head()
```

```python
num = len(bridges)-1

for i in tqdm(range(num), total=num):
    if bridges["road"][i] != bridges["road"][i+1]:
        bridges["dist"][i] = np.NaN
        bridges["verification"][i+1] = np.NaN
#         bridges['dist'][i] = geodesic(bridges["coordinates"][i], bridges["coordinates"][i+1]).kilometers
#         bridges['verification'][i+1] = bridges["km"][i] + bridges["dist"][i]
    else:
        continue
```

*Save data file.*

```python
bridges.to_excel("DataSource\\bridges_cleaned.xlsx")
```

### Data Correcting

```python
# bridges = pd.read_excel(data_path + "bridges_cleaned.xlsx")
```

**Visualize how some bridges are deviating....... explain**

```python
px.scatter(bridges[bridges['road'] == "N1"], x="km", y="verification", hover_name="LRPName")
```

```python
bridges["deviation"] = abs(bridges["km"]-bridges["verification"])
```

```python
px.scatter(bridges, x = "road", y = "deviation", hover_name = bridges.index)
```

```python
a = pd.concat([bridges, bridges.shift(+1).add_suffix('1')],
                         axis=1)[["lat", "lon", "lat1", "lon1", "dist1",
                                  "structureNr", "deviation"]]

bridges_self = pd.concat([a, bridges.shift(+2).add_suffix('2')],
                         axis=1)[["lat", "lon", "lat1", "lon1", "dist1", "lat2", "lon2",
                                  "structureNr", "deviation"]]
```

```python
bridges_self = bridges_self[bridges_self["deviation"]>25]
```

```python
bridges_self.head()
```

```python
def reverse_calc (row):
    geod = Geodesic.WGS84.Inverse(row["lat"], row["lon"], row["lat2"],row["lon2"])
    line = Geodesic.WGS84.Line(geod['lat1'],geod['lon1'], geod['azi1'])
    return line.Position(row["dist1"])["lat2"], line.Position(row["dist1"])["lon2"]
```

```python
bridges_self["correctLat"] = 0.00
bridges_self["correctLon"] = 0.00
```

```python
amendedLatLon = bridges_self.apply(lambda row: reverse_calc(row), axis=1)
```

```python
for i in tqdm(amendedLatLon.index, total=len(amendedLatLon.index)):
    bridges_self["correctLat"][i] = amendedLatLon[i][0]
    bridges_self["correctLon"][i] = amendedLatLon[i][1]
```

```python
bridges_self.head()
```

```python
bridges = pd.merge(bridges, bridges_self[["correctLat", "correctLon", "structureNr"]], on = "structureNr", how = "left")
```

```python
bridges[["correctLat", "correctLon"]] = bridges[["correctLat", "correctLon"]].fillna(0)
```

*Save data file.*

```python
bridges.to_excel("DataSource\\bridges_cleaned.xlsx")
```

```python
bridges['lat'] = np.where(bridges['correctLat']==0, bridges['lat'], bridges['correctLat'])
bridges['lon'] = np.where(bridges['correctLon']==0, bridges['lon'], bridges['correctLon'])
```

### Check if the changed coordinates have corrected our errors by calculating new_dist and new_verification

```python
bridges_self = pd.concat([bridges, bridges.shift(-1).add_suffix('2')],
                         axis=1)[["lat", "lon", "lat2", "lon2", "road", "road2", "structureNr"]]
```

```python
bridges_self = bridges_self[bridges_self.road==bridges_self.road2]
```

```python
bridges_self['new_dist'] = bridges_self.apply (lambda row: distance_calc (row),axis=1)
```

```python
bridges_CHECK = pd.merge(bridges, bridges_self[["new_dist", "structureNr"]], on = "structureNr", how = "left")
```

```python
bridges_CHECK["new_verification"] = bridges_CHECK["km"] + bridges_CHECK["new_dist"]
bridges_CHECK["new_verification"] = bridges_CHECK["new_verification"].shift(1)
```

```python
bridges_CHECK.columns
```

```python
bridges_CHECK[bridges_CHECK["deviation"]>50][["road", "km", "dist", "verification", "new_dist", "new_verification"]].head()
```

### Seems like errors are unfixed. Send helpppp..............

```python
token = 'pk.eyJ1IjoianJ5YXA5IiwiYSI6ImNrMmJ6Y216ZDAwc3UzaHFiZjdwZzJuZWIifQ.7JqX5w0UIOMnHx7jNRC1Ag'
px.set_mapbox_access_token(token)
```

```python
fig = px.scatter_mapbox(bridges, lat="lat", lon="lon", text="road", color="condition")
fig.show()
```

```python

```
