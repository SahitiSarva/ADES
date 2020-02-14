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
```

### Data Loading
**Establish current file directory and read-in data file provided. Enter bridges file name as "data_file"**

```python
data_path = os.getcwd() + "\\DataSource\\"
```

```python
data_file = "BMMS_overview.xlsx"
```

```python
bridges = pd.read_excel(data_path + data_file)
```

Potential Errors -
1. Duplicate latitude and longitude 
2. Zero latitude and longitude 

Fixed Errors - 
1. Exchanged latitudes and longotudes
2. Replaced Nan latitudes and longitudes with zeroes

Possible approaches - 
1. Check the difference in latitude and longitude and it must be meaningful 
2. Check the latitude difference with the chainage of the bridge part that is calculated. 
3. No IDs for some of the bridges (Column: name)
4. Is it true that one road has one name?


### Data Cleaning
**Some bridges had latitude and longitudes flipped. This is addressed as follows:**

```python
for i in range(len(bridges)):
    if bridges['lat'][i]>90:
        c = bridges['lat'][i] 
        bridges['lat'][i] = bridges['lon'][i] 
        bridges['lon'][i] = c
```

**Some bridges have no latitude or longitude information. For the time being, we fill these empty cells with 0.**

```python
bridges['lat'] = bridges['lat'].fillna(0)
bridges['lon'] = bridges['lon'].fillna(0)
```

### Data Fudging

**Thought Process**:
1. The BMMS_overview dataset has multiple line items with the same ID ("road" column), which we assume to be a single connected bridge.
2. It appears that the length of each bridge line item is approximately represented by the distance between its lat-long coordinates and that of the subsequent item.
3. We observed that the length of a bridge is approximately equal to the preceding line item's length + the distance between that bridge and the line item that precedes it. Thus, we calculate the distances between each line item and its preceding line item. If the numbers do not approximately add up, we assume there is an error in the coordinates.

```python
bridges["coordinates"] = np.zeros

for i in tqdm(range(len(bridges)), total=len(bridges)):
    bridges["coordinates"][i] = (bridges["lat"][i], bridges["lon"][i])
```

```python
bridges['dist'] = 0.000
```

```python
num = len(bridges)-1

for i in tqdm(range(num), total=num):
    if bridges["road"][i] == bridges["road"][i+1]:
        bridges['dist'][i] = geodesic(bridges["coordinates"][i], bridges["coordinates"][i+1]).kilometers
    else:
        continue
```

```python
bridges["verification"] = 0.00
```

```python
for i in tqdm(range(num), total=num):
    if bridges["road"][i] == bridges["road"][i+1]:
        bridges['verification'][i+1] = bridges["km"][i] + bridges["dist"][i]
    else:
        continue
```

*Save data file.*

```python
bridges.to_excel("DataSource\\bridges_cleaned.xlsx")
```

```python
bridges[["road", "lat", "lon", "length", "km", "dist", "verification"]]
```

```python

```
