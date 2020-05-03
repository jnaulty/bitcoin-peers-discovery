# Investigating Bitcoin Peer Data with bitnode.io and Ipython Notebooks

Author: John Naulty Jr.



```python
import pprint
import json

import requests
import pandas as pd
import geopandas
import matplotlib.pyplot as plt

pp = pprint.PrettyPrinter(indent=4)
```

### Interfacing with the bitnode api


```python
# https://bitnodes.io/api/


def make_request(url):
    r = requests.get(url)
    if r.status_code == 200:
        return r.json()
    return False

def write_json(file_name, json_data):
    """
    store responses in json files
    """
    with open(file_name, 'w', encoding='utf-8') as f:
        json.dump(json_data, f, ensure_ascii=False, indent=4)
        
def load_json(file_name):
    with open(file_name) as json_file:
        data = json.load(json_file)
        return data
    
    

def list_snapshot(save_response=False):
    """https://bitnodes.io/api/v1/snapshots/"""
    snapshot_url = "https://bitnodes.io/api/v1/snapshots/"
    snapshots = make_request(snapshot_url)
    pp.pprint(snapshots)
    if save_response:
        count = snapshots["count"]
        file_name = f"{count}.snapshots.json"
        print(f"writing file to: {file_name}")
        write_json(file_name, snapshots)
        

def get_nodes(snapshot_timestamp="latest", save_response=False):
    """
    https://bitnodes.io/api/v1/snapshots/<TIMESTAMP>/
    <TIMESTAMP> can be replaced with "latest" to get the list of all reachable nodes from the latest snapshot.

    """
    nodes_url = f"https://bitnodes.io/api/v1/snapshots/{snapshot_timestamp}"
    nodes = make_request(nodes_url)
    
    pp.pprint(nodes)
    if save_response:
        height = nodes["latest_height"]
        file_name = f"{height}.{snapshot_timestamp}.nodes.json"
        print(f"writing file to: {file_name}")
        write_json(file_name, nodes)
    return nodes

def tx_data_propagation(transaction):
    """
    https://bitnodes.io/api/#data-propagation
    https://bitnodes.io/api/v1/inv/<INV_HASH>/
    Values in stats represent the following information:

    head - Arrival times for the first 10 nodes in a list of ["<ADDRESS>:<PORT>", <TIMESTAMP>].
    min - Delta for earliest arrival time. Value can be 0 if the delta is less than 1 millisecond.
    max - Delta for latest arrival time.
    mean - Average of deltas.
    std - Standard deviation of deltas.
    50% - 50th percentile of deltas.
    90% - 90th percentile of deltas.
    """
    pass

```

### Saving Snapshot Data to Disk


```python
# list_snapshot(save_response=True)
# nodes = get_nodes(save_response=True)
```

### Loading Snapshot Data from Disk


```python
nodes = load_json("628680.latest.nodes.json")
```

## Data Processing with Pandas


```python


headers = [
    "Protocol version",
    "User agent",
    "Connected since",
    "Services",
    "Height",
    "Hostname",
    "City",
    "Country code",
    "Latitude",
    "Longitude",
    "Timezone",
    "ASN",
    "Organization name"
]

node_df = pd.DataFrame.from_dict(nodes['nodes'], orient='index',columns=headers)
node_df = node_df.fillna('')


node_df[["User agent", "Country code", "Organization name" ]].describe()
org_name = node_df["Organization name"]

print(node_df[["Country code", "Organization name"]].describe())
print()
print(org_name.value_counts(normalize=True).nlargest(10))

```

           Country code Organization name
    count          9234              9234
    unique           96              1034
    top              US       Tor network
    freq           1854              1544
    
    Tor network                          0.167208
    Hetzner Online GmbH                  0.110461
    Amazon.com, Inc.                     0.076998
    DigitalOcean, LLC                    0.052415
    OVH SAS                              0.051115
    Comcast Cable Communications, LLC    0.021009
    Contabo GmbH                         0.019710
    Google LLC                           0.018302
    Alibaba (US) Technology Co., Ltd.    0.014620
    Charter Communications Inc           0.013537
    Name: Organization name, dtype: float64



```python
org_name.value_counts(normalize=True).nlargest(10).plot.bar()
```




    <matplotlib.axes._subplots.AxesSubplot at 0x77d845278c18>




![png](output_10_1.png)



```python
gdf = geopandas.GeoDataFrame(
    node_df, crs="epsg:4269", geometry=geopandas.points_from_xy(node_df.Longitude, node_df.Latitude))
gdf = gdf.fillna('')
#gdf.geometry.head()
```


```python
world = geopandas.read_file(geopandas.datasets.get_path('naturalearth_lowres'))
world = world[~world.continent.isin(['Antarctica'])]

# We restrict to South America.
ax = world.plot(color='lightgrey', linewidth=1, edgecolor='black', figsize=(150,50))

# We can now plot our ``GeoDataFrame``.
gdf.plot(markersize=100, color='blue', alpha=0.5, ax=ax)
ax.axis('off')
plt.show()

```


![png](output_12_0.png)



```python
states = geopandas.read_file("shape-data/cb_2018_us_state_500k/cb_2018_us_state_500k.shp")
# Get rid of Guam, Mariana Islands and Virgin Islands
states = states[states.STATEFP.astype(int) < 60]
# Get rid of Hawaii and Alaska
states = states[~states.NAME.isin(['Hawaii', 'Alaska'])]
#states.tail(5)
```


```python
ax = states.plot(color='lightgrey', linewidth=1, edgecolor='black', figsize=(150,50))

# We can now plot our ``GeoDataFrame``.
#gdf.plot(markersize=10, color='pink', alpha=0.5, ax=ax)

us_intersection = geopandas.overlay(gdf, states, how='intersection')
us_intersection.plot(markersize=1000, color='blue', alpha=0.5, ax=ax)
ax.axis('off')
plt.show()
```


![png](output_14_0.png)



```python
gdf = gdf.fillna('')
comcast_nodes = gdf[gdf["Organization name"].str.contains('Comcast')]
comcast_intersection = geopandas.overlay(comcast_nodes, states, how='intersection')
ax = states.plot(color='lightgrey', linewidth=0.5, edgecolor='black', figsize=(150,50))
comcast_intersection.plot(markersize=1000, color='blue', alpha=0.5, ax=ax)
ax.axis('off')
plt.show()
```


![png](output_15_0.png)



```python
us_intersection["Organization name"].value_counts(normalize=True).nlargest(5).plot.bar()
```




    <matplotlib.axes._subplots.AxesSubplot at 0x77d844a9c5c0>




![png](output_16_1.png)



```python
us_intersection["NAME"].value_counts(normalize=True).nlargest(10)
```




    California    0.199674
    Virginia      0.105550
    New Jersey    0.088139
    Kansas        0.080522
    Ohio          0.065832
    Oregon        0.057671
    Texas         0.054407
    New York      0.044614
    Washington    0.026115
    Florida       0.024483
    Name: NAME, dtype: float64




```python
us_intersection[["NAME", "Organization name"]].groupby('NAME').describe()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead tr th {
        text-align: left;
    }

    .dataframe thead tr:last-of-type th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr>
      <th></th>
      <th colspan="4" halign="left">Organization name</th>
    </tr>
    <tr>
      <th></th>
      <th>count</th>
      <th>unique</th>
      <th>top</th>
      <th>freq</th>
    </tr>
    <tr>
      <th>NAME</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Alabama</th>
      <td>11</td>
      <td>5</td>
      <td>WideOpenWest Finance LLC</td>
      <td>4</td>
    </tr>
    <tr>
      <th>Arizona</th>
      <td>27</td>
      <td>12</td>
      <td>Cox Communications Inc.</td>
      <td>9</td>
    </tr>
    <tr>
      <th>Arkansas</th>
      <td>1</td>
      <td>1</td>
      <td>Comcast Cable Communications, LLC</td>
      <td>1</td>
    </tr>
    <tr>
      <th>California</th>
      <td>367</td>
      <td>43</td>
      <td>Google LLC</td>
      <td>71</td>
    </tr>
    <tr>
      <th>Colorado</th>
      <td>25</td>
      <td>7</td>
      <td>Comcast Cable Communications, LLC</td>
      <td>13</td>
    </tr>
    <tr>
      <th>Connecticut</th>
      <td>10</td>
      <td>5</td>
      <td>Cablevision Systems Corp.</td>
      <td>3</td>
    </tr>
    <tr>
      <th>District of Columbia</th>
      <td>5</td>
      <td>4</td>
      <td>MCI Communications Services, Inc. d/b/a Verizo...</td>
      <td>2</td>
    </tr>
    <tr>
      <th>Florida</th>
      <td>45</td>
      <td>14</td>
      <td>Charter Communications, Inc</td>
      <td>11</td>
    </tr>
    <tr>
      <th>Georgia</th>
      <td>32</td>
      <td>8</td>
      <td>AT&amp;T Services, Inc.</td>
      <td>13</td>
    </tr>
    <tr>
      <th>Idaho</th>
      <td>5</td>
      <td>3</td>
      <td>Cable One</td>
      <td>2</td>
    </tr>
    <tr>
      <th>Illinois</th>
      <td>30</td>
      <td>11</td>
      <td>Comcast Cable Communications, LLC</td>
      <td>15</td>
    </tr>
    <tr>
      <th>Indiana</th>
      <td>4</td>
      <td>3</td>
      <td>Comcast Cable Communications, LLC</td>
      <td>2</td>
    </tr>
    <tr>
      <th>Iowa</th>
      <td>6</td>
      <td>5</td>
      <td>Mediacom Communications Corp</td>
      <td>2</td>
    </tr>
    <tr>
      <th>Kansas</th>
      <td>148</td>
      <td>50</td>
      <td>Hurricane Electric LLC</td>
      <td>22</td>
    </tr>
    <tr>
      <th>Kentucky</th>
      <td>2</td>
      <td>2</td>
      <td>Charter Communications Inc</td>
      <td>1</td>
    </tr>
    <tr>
      <th>Louisiana</th>
      <td>1</td>
      <td>1</td>
      <td>AT&amp;T Services, Inc.</td>
      <td>1</td>
    </tr>
    <tr>
      <th>Maine</th>
      <td>5</td>
      <td>2</td>
      <td>Charter Communications Inc</td>
      <td>4</td>
    </tr>
    <tr>
      <th>Maryland</th>
      <td>31</td>
      <td>5</td>
      <td>MCI Communications Services, Inc. d/b/a Verizo...</td>
      <td>17</td>
    </tr>
    <tr>
      <th>Massachusetts</th>
      <td>37</td>
      <td>8</td>
      <td>Boston University</td>
      <td>13</td>
    </tr>
    <tr>
      <th>Michigan</th>
      <td>23</td>
      <td>6</td>
      <td>Comcast Cable Communications, LLC</td>
      <td>12</td>
    </tr>
    <tr>
      <th>Minnesota</th>
      <td>13</td>
      <td>7</td>
      <td>Comcast Cable Communications, LLC</td>
      <td>4</td>
    </tr>
    <tr>
      <th>Mississippi</th>
      <td>1</td>
      <td>1</td>
      <td>BCI Mississippi Broadband,LLC</td>
      <td>1</td>
    </tr>
    <tr>
      <th>Missouri</th>
      <td>35</td>
      <td>10</td>
      <td>Charter Communications</td>
      <td>8</td>
    </tr>
    <tr>
      <th>Montana</th>
      <td>2</td>
      <td>2</td>
      <td>Charter Communications</td>
      <td>1</td>
    </tr>
    <tr>
      <th>Nebraska</th>
      <td>2</td>
      <td>2</td>
      <td>Cox Communications Inc.</td>
      <td>1</td>
    </tr>
    <tr>
      <th>Nevada</th>
      <td>21</td>
      <td>10</td>
      <td>Cox Communications Inc.</td>
      <td>8</td>
    </tr>
    <tr>
      <th>New Hampshire</th>
      <td>10</td>
      <td>3</td>
      <td>Comcast Cable Communications, LLC</td>
      <td>8</td>
    </tr>
    <tr>
      <th>New Jersey</th>
      <td>162</td>
      <td>8</td>
      <td>DigitalOcean, LLC</td>
      <td>128</td>
    </tr>
    <tr>
      <th>New Mexico</th>
      <td>2</td>
      <td>2</td>
      <td>TULAROSA COMMUNICATIONS, INC.</td>
      <td>1</td>
    </tr>
    <tr>
      <th>New York</th>
      <td>82</td>
      <td>19</td>
      <td>Charter Communications Inc</td>
      <td>20</td>
    </tr>
    <tr>
      <th>North Carolina</th>
      <td>26</td>
      <td>5</td>
      <td>Charter Communications Inc</td>
      <td>11</td>
    </tr>
    <tr>
      <th>North Dakota</th>
      <td>1</td>
      <td>1</td>
      <td>Cable One</td>
      <td>1</td>
    </tr>
    <tr>
      <th>Ohio</th>
      <td>121</td>
      <td>8</td>
      <td>Amazon.com, Inc.</td>
      <td>96</td>
    </tr>
    <tr>
      <th>Oklahoma</th>
      <td>6</td>
      <td>3</td>
      <td>Cox Communications Inc.</td>
      <td>3</td>
    </tr>
    <tr>
      <th>Oregon</th>
      <td>106</td>
      <td>10</td>
      <td>Amazon.com, Inc.</td>
      <td>86</td>
    </tr>
    <tr>
      <th>Pennsylvania</th>
      <td>38</td>
      <td>12</td>
      <td>MCI Communications Services, Inc. d/b/a Verizo...</td>
      <td>15</td>
    </tr>
    <tr>
      <th>Rhode Island</th>
      <td>4</td>
      <td>2</td>
      <td>Cox Communications Inc.</td>
      <td>3</td>
    </tr>
    <tr>
      <th>South Carolina</th>
      <td>14</td>
      <td>9</td>
      <td>Charter Communications Inc</td>
      <td>3</td>
    </tr>
    <tr>
      <th>Tennessee</th>
      <td>7</td>
      <td>5</td>
      <td>Charter Communications</td>
      <td>2</td>
    </tr>
    <tr>
      <th>Texas</th>
      <td>100</td>
      <td>22</td>
      <td>Charter Communications Inc</td>
      <td>24</td>
    </tr>
    <tr>
      <th>Utah</th>
      <td>10</td>
      <td>4</td>
      <td>Unified Layer</td>
      <td>5</td>
    </tr>
    <tr>
      <th>Vermont</th>
      <td>3</td>
      <td>3</td>
      <td>Comcast Cable Communications, LLC</td>
      <td>1</td>
    </tr>
    <tr>
      <th>Virginia</th>
      <td>194</td>
      <td>12</td>
      <td>Amazon.com, Inc.</td>
      <td>104</td>
    </tr>
    <tr>
      <th>Washington</th>
      <td>48</td>
      <td>13</td>
      <td>Comcast Cable Communications, LLC</td>
      <td>21</td>
    </tr>
    <tr>
      <th>West Virginia</th>
      <td>1</td>
      <td>1</td>
      <td>Reliable Hosting Services</td>
      <td>1</td>
    </tr>
    <tr>
      <th>Wisconsin</th>
      <td>11</td>
      <td>5</td>
      <td>Charter Communications Inc</td>
      <td>6</td>
    </tr>
    <tr>
      <th>Wyoming</th>
      <td>3</td>
      <td>2</td>
      <td>Charter Communications</td>
      <td>2</td>
    </tr>
  </tbody>
</table>
</div>




```python
comcast_intersection["NAME"].value_counts(normalize=True).nlargest(10).plot.bar()
```




    <matplotlib.axes._subplots.AxesSubplot at 0x77d844a3d710>




![png](output_19_1.png)



```python

```
