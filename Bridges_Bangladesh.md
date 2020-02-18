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

```python
bridges.shape
```

How many road name-LRP combinations have different structure numbers?

```python
a = bridges.groupby(['road', 'LRPName', 'km', 'zone', "division", "sub-division", "circle"]).agg({'structureNr': 'count'}).reset_index()
print("Number of road name-LRP combinations with different structure numbers:",
      len(a[a['structureNr']>1]))
```

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
bridges.shape
```

```python
bridges = bridges.sort_values(["road","km"]).reset_index(drop=True)
```

```python
bridges.dtypes
```

Find and drop all duplicates that have the same road name, LRPName, and "km" value

```python
def longeststring(s):
    return max(s, key=len)
```

```python
bridges['name'] = bridges['name'].astype(str)
bridges['roadName'] = bridges['roadName'].astype(str)
```

```python
grouped_bridges = bridges.groupby(['road', 'LRPName', 'km', 'zone', "division", "sub-division", "type", "circle"]).agg({'roadName': (lambda s: longeststring(s)), 'name': (lambda s: longeststring(s)), 'length': 'max', 'structureNr':'max'
                                                                                                                , 'chainage': 'max', 'width':'max', 'constructionYear':'max', 'spans': 'max', 'lat':'max', 'lon':'max'
                                                                                                                , 'EstimatedLoc': 'max', 'condition' : 'max'})
```

```python
grouped_bridges = grouped_bridges.reset_index()
```

```python
grouped_bridges.shape
```

```python
print("Number of dropped duplicates:", bridges.shape[0] - grouped_bridges.shape[0])
```

### Data Fudging

**Thought Process**:
1. The BMMS_overview dataset has multiple line items with the same ID ("road" column), which we assume to be a single connected bridge.
2. It appears that the length of each bridge line item is approximately represented by the distance between its lat-long coordinates and that of the subsequent item.
3. We observed that the length of a bridge is approximately equal to the preceding line item's length + the distance between that bridge and the line item that precedes it. Thus, we calculate the distances between each line item and its preceding line item. If the numbers do not approximately add up, we assume there is an error in the coordinates.

```python
bridges_self = pd.concat([grouped_bridges, grouped_bridges.shift(-1).add_suffix('2')],
                         axis=1)[["lat", "lon", "lat2", "lon2", "road", "road2", "structureNr", "km"]]
```

```python
bridges_self = bridges_self[bridges_self.road==bridges_self.road2]
bridges_self = bridges_self.reset_index()
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
bridges_self["verification"] = bridges_self["km"] + bridges_self["dist"]
bridges_self["verification"] = bridges_self["verification"].shift(1)
```

```python
grouped_bridges = pd.merge(grouped_bridges, bridges_self[["dist", "structureNr", "verification"]], on = "structureNr", how = "left")
```

### Data Correcting


**Visualize how some bridges are deviating. Explain**

```python
px.scatter(grouped_bridges[grouped_bridges['road'] == "N1"], x="km", y="verification", hover_name="LRPName")
```

```python
grouped_bridges["deviation"] = abs(grouped_bridges["km"]-grouped_bridges["verification"])
```

**However**, if there are 3 points in a row (e.g., points A, B, and C, in the order of increasing chainage) are wrongly placed with approximately the same magnitude of error, then the deviation/verification of point B will not reflect an error. This is because the distance from point B to point A are both calculated off wrong points. The verification/deviations calculated for point B will not reveal an error. Thus, we identify the points where points A and C have high deviations, but where point B is noted to have a low deviation value, and we replace the point B deviation with the point A deviation (an approximation). 

```python
num = len(grouped_bridges)-1
```

```python
id_fix=[]

for i in tqdm(range(num), total = num):
    try:
        if (grouped_bridges["deviation"][i] >5) & (grouped_bridges["deviation"][i+2] > 5) & (grouped_bridges["deviation"][i+1] <= 5):
            id_fix.append(i+1)
    except KeyError:
        continue
```

```python
len(id_fix)
```

```python
for i in id_fix:
    grouped_bridges.deviation[i] = grouped_bridges.deviation[i-1]
```

```python
px.scatter(grouped_bridges, x = "road", y = "deviation", hover_name = grouped_bridges.index)
```

We correct the row items with large deviations by approximating the correct coordinates based on difference in chainage (assumed to be correct distance between one row and the next) and the previous two coordinates.

```python
for i in tqdm(range(num), total=num):
    try:
        if grouped_bridges["road"][i] == grouped_bridges["road"][i-1]:
            if grouped_bridges.deviation[i] > 5:
                geod = Geodesic.WGS84.Inverse(grouped_bridges["lat"][i-2], grouped_bridges["lon"][i-2],
                                              grouped_bridges["lat"][i-1], grouped_bridges["lon"][i-1])
                line = Geodesic.WGS84.Line(geod['lat1'],geod['lon1'], geod['azi1'])
    #             print(i, "old      ", grouped_bridges["lat"][i], grouped_bridges["lon"][i])
    #             print(grouped_bridges["km"][i-1]-grouped_bridges["km"][i])
                grouped_bridges["lat"][i] = line.Position(grouped_bridges["km"][i]-grouped_bridges["km"][i-2])["lat2"]
                grouped_bridges["lon"][i] = line.Position(grouped_bridges["km"][i]-grouped_bridges["km"][i-2])["lon2"]
    #             print(i, "corrected", grouped_bridges["lat"][i], grouped_bridges["lon"][i])
        else:
            continue
    except KeyError:
        continue
```

```python
token = "pk.eyJ1Ijoic2FoaXRpcyIsImEiOiJjazZtY3FvYzEwbWs4M2xsc25nOW1ndmo5In0.EWX1gLsUKxAIGorGR4czuQ"
px.set_mapbox_access_token(token)
```

```python
px.scatter_mapbox(grouped_bridges, lat ="lat", lon="lon", hover_name = grouped_bridges.index)
```

Amend the dataframe with corrected coordinates (grouped_bridges) such that it matches the dataframe (in terms of datatypes) of the original dataframe from the BMMS_overview.xlsx file.

```python
bridges_ori = pd.read_excel(data_path + data_file)
```

```python
col_names = list(bridges_ori.columns)
col_names.extend(["dist", "verification", "deviation"])
```

```python
grouped_bridges = grouped_bridges[col_names]
```

```python
del grouped_bridges["deviation"]
```

```python
del grouped_bridges["dist"]
```

```python
del grouped_bridges["verification"]
```

```python
for x in grouped_bridges.columns:
    grouped_bridges[x]=grouped_bridges[x].astype(bridges_ori[x].dtypes.name)
```

*Save data file, match original BMMS_overview.xlsx file format (e.g., sheet name).*

```python
data_path
```

```python
grouped_bridges.set_index("road").to_excel(data_path + "BMMS_overview.xlsx", sheet_name="BMMS_overview")
```

# Done!
