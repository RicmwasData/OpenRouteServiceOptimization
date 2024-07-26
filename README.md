# Route Optimization with ORS
In this project I cover vehicle Route optimization with <a href="https://openrouteservice.org/">Openrouteservice API.</a> OpenRouteService (ORS) is an open-source routing service provided by HeiGIT (Heidelberg Institute for Geoinformation Technology). It offers various geospatial services through a set of web-based APIs. 
# Project Overview

## Project Aim
The aim of this project is to optimize the delivery routes for three vehicles tasked with making deliveries to 30 randomly selected pubs in the UK. These pubs are chosen from a dataset of 50,000 pubs provided by <a href="https://www.kaggle.com/datasets/rtatman/every-pub-in-england?select=open_pubs.csv/">the Open Pubs Data on Kaggle.</a> 

It is importatnt to note that Openrouteservice relays on <a href="https://github.com/VROOM-Project/vroom/blob/master/docs/API.md">Vroom API.</a>

## Optimization Criteria
1. **Service Time**: Each vehicle will spend a number of minutes at each delivery station, calculated by multiplying the amount by 10 seconds.
2. **Operating Hours**: Each pub opens between 7:00 AM and 9:00 AM and closes between 5:00 PM and 8:00 PM. Deliveries will occur between 7:00 AM and 8:00 PM.
3. **Return to Depot**: All deliveries start from the depot, and vehicles must return to the depot by the end of the day.
4. **Vehicle Limitations**: The OpenRouteService API restricts the optimization to a maximum of three vehicles.
5. **Random Selection**: Pubs are selected randomly for each run, resulting in different routes for each execution.
6. **Time Window Constraint**: Due to time window constraints, the three vehicles may not be able to deliver to all 30 stations within the specified timeframe.

## Final Output
1. **Dataframe**: A dataframe showing the time taken by each vehicle in seconds and the distance covered in kilometers.
2. **Route Map**: A map displaying the route, arrival time at each station, and departure time from each station.



## import libraries 


```python
import pandas as pd
import leafmap.kepler as leafmap
import geopandas as gpd
import random
import folium
import openrouteservice as ors
import numpy as np
import os
import datetime
```

## Load the england pubs data 


```python
df= pd.read_csv("open_pubs.csv")
df['longitude']= pd.to_numeric(df['longitude'], errors='coerce')
df['latitude']= pd.to_numeric(df['latitude'],errors='coerce')
df['Title']= "Pub"
df['Amount']= np.random.randint(10, 201, df.shape[0])
df['Service']= df['Amount']*10
df['Service_hours']=  list(map(lambda x: str(datetime.timedelta(seconds=x)), df['Service']))
df.dropna(inplace=True)
df= df.sample(n=30, random_state=1)
df.reset_index(drop=True, inplace=True)

# Generate random times between 7 AM and 9 AM on 25/07/2024
date = "2024-07-25"
Open_start_time = pd.Timestamp(f"{date} 07:00:00")
Open_end_time = pd.Timestamp(f"{date} 09:00:00")

Close_start_time = pd.Timestamp(f"{date} 17:00:00")
Close_end_time = pd.Timestamp(f"{date} 20:00:00")


# Generate random times within the specified range
df['Open_Time'] = [Open_start_time + pd.Timedelta(minutes=np.random.randint(0, 120)) for _ in range(df.shape[0])]
df['Open_Time']=df['Open_Time'].apply(lambda x: int(x.timestamp()))
df['Close_Time'] = [Close_start_time + pd.Timedelta(minutes=np.random.randint(0, 120)) for _ in range(df.shape[0])]
df['Close_Time']= df['Close_Time'].apply(lambda x: int(x.timestamp()))
df.reset_index(inplace=True)
df.head()

```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>index</th>
      <th>fas_id</th>
      <th>name</th>
      <th>address</th>
      <th>postcode</th>
      <th>easting</th>
      <th>northing</th>
      <th>latitude</th>
      <th>longitude</th>
      <th>local_authority</th>
      <th>Title</th>
      <th>Amount</th>
      <th>Service</th>
      <th>Service_hours</th>
      <th>Open_Time</th>
      <th>Close_Time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>0</td>
      <td>470543</td>
      <td>Beith Masonic Club</td>
      <td>Eglinton Street, Beith, Ayrshire</td>
      <td>KA15 1AD</td>
      <td>234854</td>
      <td>653907.0</td>
      <td>55.750284</td>
      <td>-4.632828</td>
      <td>North Ayrshire</td>
      <td>Pub</td>
      <td>176</td>
      <td>1760</td>
      <td>0:29:20</td>
      <td>1721891880</td>
      <td>1721927820</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1</td>
      <td>216765</td>
      <td>The Buck Inn</td>
      <td>59 Green Lane, Sale, Cheshire</td>
      <td>M33 5PN</td>
      <td>377347</td>
      <td>392597.0</td>
      <td>53.429668</td>
      <td>-2.342400</td>
      <td>Trafford</td>
      <td>Pub</td>
      <td>113</td>
      <td>1130</td>
      <td>0:18:50</td>
      <td>1721893920</td>
      <td>1721926800</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2</td>
      <td>415351</td>
      <td>The Three Swans</td>
      <td>23 Church Hill, Selby, North Yorkshire</td>
      <td>YO8 4PL</td>
      <td>461625</td>
      <td>432482.0</td>
      <td>53.785022</td>
      <td>-1.066165</td>
      <td>Selby</td>
      <td>Pub</td>
      <td>167</td>
      <td>1670</td>
      <td>0:27:50</td>
      <td>1721892720</td>
      <td>1721932320</td>
    </tr>
    <tr>
      <th>3</th>
      <td>3</td>
      <td>342027</td>
      <td>Greyhound Inn</td>
      <td>Court Lane, Erdington, Birmingham</td>
      <td>B23 5JX</td>
      <td>410296</td>
      <td>293685.0</td>
      <td>52.540917</td>
      <td>-1.849621</td>
      <td>Birmingham</td>
      <td>Pub</td>
      <td>94</td>
      <td>940</td>
      <td>0:15:40</td>
      <td>1721896740</td>
      <td>1721930400</td>
    </tr>
    <tr>
      <th>4</th>
      <td>4</td>
      <td>19003</td>
      <td>Green Gate Inn</td>
      <td>High Street, Caister-On-Sea, Norfolk</td>
      <td>NR30 5EL</td>
      <td>652271</td>
      <td>312029.0</td>
      <td>52.647280</td>
      <td>1.727878</td>
      <td>Great Yarmouth</td>
      <td>Pub</td>
      <td>60</td>
      <td>600</td>
      <td>0:10:00</td>
      <td>1721893620</td>
      <td>1721933220</td>
    </tr>
  </tbody>
</table>
</div>



## Openrouteservice api_key


```python
from dotenv import load_dotenv
load_dotenv()
api_key= os.getenv("api_key")
client= ors.client.Client(api_key)
```

## Createing Vehicles ,Jobs


```python
date = "2024-07-25"
start_time = pd.Timestamp(f"{date} 07:00:00")
end_time = pd.Timestamp(f"{date} 20:00:00")

# Convert to integers (seconds since epoch)
start_time_int = int(start_time.timestamp())
end_time_int = int(end_time.timestamp())
vehicle_start=  [-0.31458977680631506,52.68531116549616 ]
```


```python


vehicles=list()
for idx in range(3):
    vehicles.append(
        ors.optimization.Vehicle(
            id= idx,
            start=vehicle_start,
            end= vehicle_start,
            capacity=[30],
            time_window=[start_time_int , end_time_int ] 
        )
    )

deliveries= list()
for delivery in df.itertuples():
    deliveries.append(
        ors.optimization.Job(
            id= delivery.index,
            location=[delivery.longitude, delivery.latitude],
            service= delivery.Service,
             time_windows=[[
                delivery.Open_Time,  # VROOM expects UNIX timestamp
                delivery.Close_Time
            ]]
        )
    )
```

## Optimize the Routes


```python
optimized = client.optimization(jobs=deliveries, vehicles=vehicles, geometry=True)
line_colors = ['green', 'orange', 'blue', 'yellow']

```


```python
geometry= gpd.GeoSeries.from_xy(df.longitude, df.latitude, crs='EPSG:4326')
geo_df = gpd.GeoDataFrame(df[['name','address', 'Title']], geometry=geometry)
geo_df.dropna(inplace=True)
m = folium.Map(location=[51.51172538779912,-0.1275698006594154], zoom_start=7, tiles="cartodb positron")
folium.GeoJson(geo_df, name="Pubs",
                marker= folium.CircleMarker(radius=8,weight=2, fill_color="red", fill_opacity=1),                      
                tooltip=folium.GeoJsonTooltip(fields=["Title","name",'address']),
                popup=folium.GeoJsonPopup(fields=['Title', "name",'address']),
                highlight_function=lambda x: {"fillOpacity": 0.8},
                zoom_on_click=True
    ).add_to(m)
folium.CircleMarker(location= list(reversed(vehicle_start)),
                    tooltip='Title: Indu',
    radius=8,weight=2, fill_color="blue", fill_opacity=1).add_to(m)
coords = df.apply(lambda row: [row['longitude'], row['latitude']], axis=1).tolist()
for route in optimized['routes']:
    folium.PolyLine(locations=[list(reversed(coords)) for coords in ors.convert.decode_polyline(route['geometry'])['coordinates']], color=line_colors[route['vehicle']]).add_to(m)
m
```


## Extract more information from the Route


```python
# Only extract relevant fields from the response
extract_fields = ['distance', 'duration']
data = [{key: route[key] for key in extract_fields} for route in optimized['routes']]

vehicles_df = pd.DataFrame(data)
vehicles_df.index.name = 'vehicle'
vehicles_df
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>distance</th>
      <th>duration</th>
    </tr>
    <tr>
      <th>vehicle</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>783440</td>
      <td>36015</td>
    </tr>
    <tr>
      <th>1</th>
      <td>623860</td>
      <td>31438</td>
    </tr>
    <tr>
      <th>2</th>
      <td>779780</td>
      <td>36109</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Create a list to display the schedule for all vehicles
stations = list()
for route in optimized['routes']:
    vehicle = list()
    for step in route["steps"]:
        vehicle.append(
            [
                step.get("job", "Depot"),  # Station ID
                step["arrival"],  # Arrival time
                step["arrival"] + step.get("service", 0),  # Departure time

            ]
        )
    stations.append(vehicle)
```


```python
# Initialize an empty list to store DataFrames
df_list = []

# Loop through each vehicle's schedule and create a DataFrame
for vehicle_id, vehicle_schedule in enumerate(stations):
    _ = pd.DataFrame(vehicle_schedule, columns=['index', 'Arrival', 'Departure'])
    _['VehicleID'] = vehicle_id
    df_list.append(_)

# Concatenate all DataFrames into a single DataFrame
df_ = pd.concat(df_list, ignore_index=True)
df_ ['Arrival'] = pd.to_datetime(df_ ['Arrival'], unit='s').astype(str)
df_ ['Departure'] = pd.to_datetime(df_ ['Departure'], unit='s').astype(str)
df_ 
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>index</th>
      <th>Arrival</th>
      <th>Departure</th>
      <th>VehicleID</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Depot</td>
      <td>2024-07-25 07:00:00</td>
      <td>2024-07-25 07:00:00</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>18</td>
      <td>2024-07-25 07:36:09</td>
      <td>2024-07-25 07:55:39</td>
      <td>0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>11</td>
      <td>2024-07-25 09:34:41</td>
      <td>2024-07-25 09:38:41</td>
      <td>0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>10</td>
      <td>2024-07-25 09:59:42</td>
      <td>2024-07-25 10:28:12</td>
      <td>0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>19</td>
      <td>2024-07-25 12:48:35</td>
      <td>2024-07-25 13:20:55</td>
      <td>0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>23</td>
      <td>2024-07-25 14:39:04</td>
      <td>2024-07-25 14:59:04</td>
      <td>0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>2</td>
      <td>2024-07-25 15:57:00</td>
      <td>2024-07-25 16:24:50</td>
      <td>0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>20</td>
      <td>2024-07-25 17:05:01</td>
      <td>2024-07-25 17:14:51</td>
      <td>0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>16</td>
      <td>2024-07-25 17:44:46</td>
      <td>2024-07-25 18:02:16</td>
      <td>0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>Depot</td>
      <td>2024-07-25 19:39:45</td>
      <td>2024-07-25 19:39:45</td>
      <td>0</td>
    </tr>
    <tr>
      <th>10</th>
      <td>Depot</td>
      <td>2024-07-25 07:00:00</td>
      <td>2024-07-25 07:00:00</td>
      <td>1</td>
    </tr>
    <tr>
      <th>11</th>
      <td>6</td>
      <td>2024-07-25 08:51:42</td>
      <td>2024-07-25 09:24:22</td>
      <td>1</td>
    </tr>
    <tr>
      <th>12</th>
      <td>24</td>
      <td>2024-07-25 10:27:25</td>
      <td>2024-07-25 10:54:05</td>
      <td>1</td>
    </tr>
    <tr>
      <th>13</th>
      <td>29</td>
      <td>2024-07-25 11:06:31</td>
      <td>2024-07-25 11:36:51</td>
      <td>1</td>
    </tr>
    <tr>
      <th>14</th>
      <td>25</td>
      <td>2024-07-25 12:18:17</td>
      <td>2024-07-25 12:35:47</td>
      <td>1</td>
    </tr>
    <tr>
      <th>15</th>
      <td>15</td>
      <td>2024-07-25 13:29:18</td>
      <td>2024-07-25 13:58:48</td>
      <td>1</td>
    </tr>
    <tr>
      <th>16</th>
      <td>28</td>
      <td>2024-07-25 14:32:46</td>
      <td>2024-07-25 14:50:46</td>
      <td>1</td>
    </tr>
    <tr>
      <th>17</th>
      <td>14</td>
      <td>2024-07-25 15:28:53</td>
      <td>2024-07-25 15:52:53</td>
      <td>1</td>
    </tr>
    <tr>
      <th>18</th>
      <td>17</td>
      <td>2024-07-25 16:51:33</td>
      <td>2024-07-25 17:24:23</td>
      <td>1</td>
    </tr>
    <tr>
      <th>19</th>
      <td>Depot</td>
      <td>2024-07-25 19:15:28</td>
      <td>2024-07-25 19:15:28</td>
      <td>1</td>
    </tr>
    <tr>
      <th>20</th>
      <td>Depot</td>
      <td>2024-07-25 07:00:00</td>
      <td>2024-07-25 07:00:00</td>
      <td>2</td>
    </tr>
    <tr>
      <th>21</th>
      <td>3</td>
      <td>2024-07-25 08:39:26</td>
      <td>2024-07-25 08:55:06</td>
      <td>2</td>
    </tr>
    <tr>
      <th>22</th>
      <td>12</td>
      <td>2024-07-25 09:33:57</td>
      <td>2024-07-25 09:49:27</td>
      <td>2</td>
    </tr>
    <tr>
      <th>23</th>
      <td>7</td>
      <td>2024-07-25 11:04:31</td>
      <td>2024-07-25 11:19:31</td>
      <td>2</td>
    </tr>
    <tr>
      <th>24</th>
      <td>8</td>
      <td>2024-07-25 12:09:06</td>
      <td>2024-07-25 12:39:06</td>
      <td>2</td>
    </tr>
    <tr>
      <th>25</th>
      <td>1</td>
      <td>2024-07-25 12:47:54</td>
      <td>2024-07-25 13:06:44</td>
      <td>2</td>
    </tr>
    <tr>
      <th>26</th>
      <td>27</td>
      <td>2024-07-25 14:15:27</td>
      <td>2024-07-25 14:19:37</td>
      <td>2</td>
    </tr>
    <tr>
      <th>27</th>
      <td>21</td>
      <td>2024-07-25 15:23:49</td>
      <td>2024-07-25 15:30:29</td>
      <td>2</td>
    </tr>
    <tr>
      <th>28</th>
      <td>5</td>
      <td>2024-07-25 16:15:04</td>
      <td>2024-07-25 16:44:14</td>
      <td>2</td>
    </tr>
    <tr>
      <th>29</th>
      <td>9</td>
      <td>2024-07-25 17:28:00</td>
      <td>2024-07-25 17:31:20</td>
      <td>2</td>
    </tr>
    <tr>
      <th>30</th>
      <td>Depot</td>
      <td>2024-07-25 19:20:09</td>
      <td>2024-07-25 19:20:09</td>
      <td>2</td>
    </tr>
  </tbody>
</table>
</div>



The data shows the stations visited by each car arrival time and departure time. Next I added the routes information to the map. 


```python
df.columns
```




    Index(['index', 'fas_id', 'name', 'address', 'postcode', 'easting', 'northing',
           'latitude', 'longitude', 'local_authority', 'Title', 'Amount',
           'Service', 'Service_hours', 'Open_Time', 'Close_Time'],
          dtype='object')




```python
# Perform inner join
final_df=pd.merge(df_ , df, on='index', how='right')
geometry= gpd.GeoSeries.from_xy(final_df.longitude, final_df.latitude, crs='EPSG:4326')
geo_df = gpd.GeoDataFrame(final_df[['name','address', 'Title',
                                    'VehicleID','Arrival','Departure','Service_hours']],
                           geometry=geometry)
m = folium.Map(location=[51.51172538779912,-0.1275698006594154], zoom_start=7, tiles="cartodb positron")
folium.GeoJson(geo_df, name="Pubs",
                marker= folium.CircleMarker(radius=8,weight=2, fill_color="red", fill_opacity=1),                      
                tooltip=folium.GeoJsonTooltip(fields=["Title","name",'address',
                                                      'VehicleID','Arrival','Departure','Service_hours']),
                popup=folium.GeoJsonPopup(fields=['Title', "name",'address',
                                                  'VehicleID','Arrival','Departure','Service_hours']),
                highlight_function=lambda x: {"fillOpacity": 0.8},
                zoom_on_click=True
    ).add_to(m)
folium.CircleMarker(location= list(reversed(vehicle_start)),
                    tooltip='Title: Indu',
    radius=8,weight=2, fill_color="blue", fill_opacity=1).add_to(m)
coords = df.apply(lambda row: [row['longitude'], row['latitude']], axis=1).tolist()
for route in optimized['routes']:
    folium.PolyLine(locations=[list(reversed(coords)) for coords in ors.convert.decode_polyline(route['geometry'])['coordinates']], color=line_colors[route['vehicle']]).add_to(m)
m
```