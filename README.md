::: {role="main"}
::: {.jp-Cell .jp-MarkdownCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedMarkdown .jp-MarkdownOutput mime-type="text/markdown"}
Route Optimization with ORS[¶](#Route-Optimization-with-ORS){.anchor-link} {#Route-Optimization-with-ORS}
==========================================================================

In this project I cover vehicle Route optimization with
[Openrouteservice API.](https://openrouteservice.org/) OpenRouteService
(ORS) is an open-source routing service provided by HeiGIT (Heidelberg
Institute for Geoinformation Technology). It offers various geospatial
services through a set of web-based APIs.

Project Overview[¶](#Project-Overview){.anchor-link} {#Project-Overview}
====================================================

Project Aim[¶](#Project-Aim){.anchor-link} {#Project-Aim}
------------------------------------------

The aim of this project is to optimize the delivery routes for three
vehicles tasked with making deliveries to 30 randomly selected pubs in
the UK. These pubs are chosen from a dataset of 50,000 pubs provided by
[the Open Pubs Data on
Kaggle.](https://www.kaggle.com/datasets/rtatman/every-pub-in-england?select=open_pubs.csv/)

It is importatnt to note that Openrouteservice relays on [Vroom
API.](https://github.com/VROOM-Project/vroom/blob/master/docs/API.md)

Optimization Criteria[¶](#Optimization-Criteria){.anchor-link} {#Optimization-Criteria}
--------------------------------------------------------------

1.  **Service Time**: Each vehicle will spend a number of minutes at
    each delivery station, calculated by multiplying the amount by 10
    seconds.
2.  **Operating Hours**: Each pub opens between 7:00 AM and 9:00 AM and
    closes between 5:00 PM and 8:00 PM. Deliveries will occur between
    7:00 AM and 8:00 PM.
3.  **Return to Depot**: All deliveries start from the depot, and
    vehicles must return to the depot by the end of the day.
4.  **Vehicle Limitations**: The OpenRouteService API restricts the
    optimization to a maximum of three vehicles.
5.  **Random Selection**: Pubs are selected randomly for each run,
    resulting in different routes for each execution.
6.  **Time Window Constraint**: Due to time window constraints, the
    three vehicles may not be able to deliver to all 30 stations within
    the specified timeframe.

Final Output[¶](#Final-Output){.anchor-link} {#Final-Output}
--------------------------------------------

1.  **Dataframe**: A dataframe showing the time taken by each vehicle in
    seconds and the distance covered in kilometers.
2.  **Route Map**: A map displaying the route, arrival time at each
    station, and departure time from each station.
:::
:::
:::
:::

::: {.jp-Cell .jp-MarkdownCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedMarkdown .jp-MarkdownOutput mime-type="text/markdown"}
import libraries[¶](#import-libraries){.anchor-link}
----------------------------------------------------
:::
:::
:::
:::

::: {.jp-Cell .jp-CodeCell .jp-Notebook-cell .jp-mod-noOutputs}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
In \[ \]:
:::

::: {.jp-CodeMirrorEditor .jp-Editor .jp-InputArea-editor data-type="inline"}
::: {.cm-editor .cm-s-jupyter}
::: {.highlight .hl-ipython3}
    import pandas as pd
    import leafmap.kepler as leafmap
    import geopandas as gpd
    import random
    import folium
    import openrouteservice as ors
    import numpy as np
    import os
    import datetime
:::
:::
:::
:::
:::
:::

::: {.jp-Cell .jp-MarkdownCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedMarkdown .jp-MarkdownOutput mime-type="text/markdown"}
Load the england pubs data[¶](#Load-the-england-pubs-data){.anchor-link} {#Load-the-england-pubs-data}
------------------------------------------------------------------------
:::
:::
:::
:::

::: {.jp-Cell .jp-CodeCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
In \[ \]:
:::

::: {.jp-CodeMirrorEditor .jp-Editor .jp-InputArea-editor data-type="inline"}
::: {.cm-editor .cm-s-jupyter}
::: {.highlight .hl-ipython3}
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
:::
:::
:::
:::
:::

::: {.jp-Cell-outputWrapper}
::: {.jp-Collapser .jp-OutputCollapser .jp-Cell-outputCollapser}
:::

::: {.jp-OutputArea .jp-Cell-outputArea}
::: {.jp-OutputArea-child .jp-OutputArea-executeResult}
::: {.jp-OutputPrompt .jp-OutputArea-prompt}
Out\[ \]:
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedHTML .jp-OutputArea-output .jp-OutputArea-executeResult mime-type="text/html" tabindex="0"}
<div>

      index   fas\_id   name                 address                                  postcode   easting   northing   latitude    longitude   local\_authority   Title   Amount   Service   Service\_hours   Open\_Time   Close\_Time
  --- ------- --------- -------------------- ---------------------------------------- ---------- --------- ---------- ----------- ----------- ------------------ ------- -------- --------- ---------------- ------------ -------------
  0   0       470543    Beith Masonic Club   Eglinton Street, Beith, Ayrshire         KA15 1AD   234854    653907.0   55.750284   -4.632828   North Ayrshire     Pub     176      1760      0:29:20          1721891880   1721927820
  1   1       216765    The Buck Inn         59 Green Lane, Sale, Cheshire            M33 5PN    377347    392597.0   53.429668   -2.342400   Trafford           Pub     113      1130      0:18:50          1721893920   1721926800
  2   2       415351    The Three Swans      23 Church Hill, Selby, North Yorkshire   YO8 4PL    461625    432482.0   53.785022   -1.066165   Selby              Pub     167      1670      0:27:50          1721892720   1721932320
  3   3       342027    Greyhound Inn        Court Lane, Erdington, Birmingham        B23 5JX    410296    293685.0   52.540917   -1.849621   Birmingham         Pub     94       940       0:15:40          1721896740   1721930400
  4   4       19003     Green Gate Inn       High Street, Caister-On-Sea, Norfolk     NR30 5EL   652271    312029.0   52.647280   1.727878    Great Yarmouth     Pub     60       600       0:10:00          1721893620   1721933220

</div>
:::
:::
:::
:::
:::

::: {.jp-Cell .jp-MarkdownCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedMarkdown .jp-MarkdownOutput mime-type="text/markdown"}
Openrouteservice api\_key[¶](#Openrouteservice-api_key){.anchor-link} {#Openrouteservice-api_key}
---------------------------------------------------------------------
:::
:::
:::
:::

::: {.jp-Cell .jp-CodeCell .jp-Notebook-cell .jp-mod-noOutputs}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
In \[ \]:
:::

::: {.jp-CodeMirrorEditor .jp-Editor .jp-InputArea-editor data-type="inline"}
::: {.cm-editor .cm-s-jupyter}
::: {.highlight .hl-ipython3}
    from dotenv import load_dotenv
    load_dotenv()
    api_key= os.getenv("api_key")
    client= ors.client.Client(api_key)
:::
:::
:::
:::
:::
:::

::: {.jp-Cell .jp-MarkdownCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedMarkdown .jp-MarkdownOutput mime-type="text/markdown"}
Createing Vehicles ,Jobs[¶](#Createing-Vehicles-,Jobs){.anchor-link} {#Createing-Vehicles-,Jobs}
--------------------------------------------------------------------
:::
:::
:::
:::

::: {.jp-Cell .jp-CodeCell .jp-Notebook-cell .jp-mod-noOutputs}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
In \[ \]:
:::

::: {.jp-CodeMirrorEditor .jp-Editor .jp-InputArea-editor data-type="inline"}
::: {.cm-editor .cm-s-jupyter}
::: {.highlight .hl-ipython3}
    date = "2024-07-25"
    start_time = pd.Timestamp(f"{date} 07:00:00")
    end_time = pd.Timestamp(f"{date} 20:00:00")

    # Convert to integers (seconds since epoch)
    start_time_int = int(start_time.timestamp())
    end_time_int = int(end_time.timestamp())
    vehicle_start=  [-0.31458977680631506,52.68531116549616 ]
:::
:::
:::
:::
:::
:::

::: {.jp-Cell .jp-CodeCell .jp-Notebook-cell .jp-mod-noOutputs}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
In \[ \]:
:::

::: {.jp-CodeMirrorEditor .jp-Editor .jp-InputArea-editor data-type="inline"}
::: {.cm-editor .cm-s-jupyter}
::: {.highlight .hl-ipython3}
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
:::
:::
:::
:::
:::
:::

::: {.jp-Cell .jp-MarkdownCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedMarkdown .jp-MarkdownOutput mime-type="text/markdown"}
Optimize the Routes[¶](#Optimize-the-Routes){.anchor-link} {#Optimize-the-Routes}
----------------------------------------------------------
:::
:::
:::
:::

::: {.jp-Cell .jp-CodeCell .jp-Notebook-cell .jp-mod-noOutputs}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
In \[ \]:
:::

::: {.jp-CodeMirrorEditor .jp-Editor .jp-InputArea-editor data-type="inline"}
::: {.cm-editor .cm-s-jupyter}
::: {.highlight .hl-ipython3}
    optimized = client.optimization(jobs=deliveries, vehicles=vehicles, geometry=True)
    line_colors = ['green', 'orange', 'blue', 'yellow']
:::
:::
:::
:::
:::
:::

::: {.jp-Cell .jp-CodeCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
In \[ \]:
:::

::: {.jp-CodeMirrorEditor .jp-Editor .jp-InputArea-editor data-type="inline"}
::: {.cm-editor .cm-s-jupyter}
::: {.highlight .hl-ipython3}
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
:::
:::
:::
:::
:::

::: {.jp-Cell-outputWrapper}
::: {.jp-Collapser .jp-OutputCollapser .jp-Cell-outputCollapser}
:::

::: {.jp-OutputArea .jp-Cell-outputArea}
::: {.jp-OutputArea-child .jp-OutputArea-executeResult}
::: {.jp-OutputPrompt .jp-OutputArea-prompt}
Out\[ \]:
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedHTML .jp-OutputArea-output .jp-OutputArea-executeResult mime-type="text/html" tabindex="0"}
::: {style="width:100%;"}
::: {style="position:relative;width:100%;height:0;padding-bottom:60%;"}
[Make this Notebook Trusted to load map: File -\> Trust
Notebook]{style="color:#565656"}
:::
:::
:::
:::
:::
:::
:::

::: {.jp-Cell .jp-MarkdownCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedMarkdown .jp-MarkdownOutput mime-type="text/markdown"}
Extract more information from the Route[¶](#Extract-more-information-from-the-Route){.anchor-link} {#Extract-more-information-from-the-Route}
--------------------------------------------------------------------------------------------------
:::
:::
:::
:::

::: {.jp-Cell .jp-CodeCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
In \[ \]:
:::

::: {.jp-CodeMirrorEditor .jp-Editor .jp-InputArea-editor data-type="inline"}
::: {.cm-editor .cm-s-jupyter}
::: {.highlight .hl-ipython3}
    # Only extract relevant fields from the response
    extract_fields = ['distance', 'duration']
    data = [{key: route[key] for key in extract_fields} for route in optimized['routes']]

    vehicles_df = pd.DataFrame(data)
    vehicles_df.index.name = 'vehicle'
    vehicles_df
:::
:::
:::
:::
:::

::: {.jp-Cell-outputWrapper}
::: {.jp-Collapser .jp-OutputCollapser .jp-Cell-outputCollapser}
:::

::: {.jp-OutputArea .jp-Cell-outputArea}
::: {.jp-OutputArea-child .jp-OutputArea-executeResult}
::: {.jp-OutputPrompt .jp-OutputArea-prompt}
Out\[ \]:
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedHTML .jp-OutputArea-output .jp-OutputArea-executeResult mime-type="text/html" tabindex="0"}
<div>

</div>
:::
:::
:::
:::
:::
:::

distance

duration

vehicle

0

783440

36015

1

623860

31438

2

779780

36109

::: {.jp-Cell .jp-CodeCell .jp-Notebook-cell .jp-mod-noOutputs}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
In \[ \]:
:::

::: {.jp-CodeMirrorEditor .jp-Editor .jp-InputArea-editor data-type="inline"}
::: {.cm-editor .cm-s-jupyter}
::: {.highlight .hl-ipython3}
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
:::
:::
:::
:::
:::
:::

::: {.jp-Cell .jp-CodeCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
In \[ \]:
:::

::: {.jp-CodeMirrorEditor .jp-Editor .jp-InputArea-editor data-type="inline"}
::: {.cm-editor .cm-s-jupyter}
::: {.highlight .hl-ipython3}
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
:::
:::
:::
:::
:::

::: {.jp-Cell-outputWrapper}
::: {.jp-Collapser .jp-OutputCollapser .jp-Cell-outputCollapser}
:::

::: {.jp-OutputArea .jp-Cell-outputArea}
::: {.jp-OutputArea-child .jp-OutputArea-executeResult}
::: {.jp-OutputPrompt .jp-OutputArea-prompt}
Out\[ \]:
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedHTML .jp-OutputArea-output .jp-OutputArea-executeResult mime-type="text/html" tabindex="0"}
<div>

       index   Arrival               Departure             VehicleID
  ---- ------- --------------------- --------------------- -----------
  0    Depot   2024-07-25 07:00:00   2024-07-25 07:00:00   0
  1    18      2024-07-25 07:36:09   2024-07-25 07:55:39   0
  2    11      2024-07-25 09:34:41   2024-07-25 09:38:41   0
  3    10      2024-07-25 09:59:42   2024-07-25 10:28:12   0
  4    19      2024-07-25 12:48:35   2024-07-25 13:20:55   0
  5    23      2024-07-25 14:39:04   2024-07-25 14:59:04   0
  6    2       2024-07-25 15:57:00   2024-07-25 16:24:50   0
  7    20      2024-07-25 17:05:01   2024-07-25 17:14:51   0
  8    16      2024-07-25 17:44:46   2024-07-25 18:02:16   0
  9    Depot   2024-07-25 19:39:45   2024-07-25 19:39:45   0
  10   Depot   2024-07-25 07:00:00   2024-07-25 07:00:00   1
  11   6       2024-07-25 08:51:42   2024-07-25 09:24:22   1
  12   24      2024-07-25 10:27:25   2024-07-25 10:54:05   1
  13   29      2024-07-25 11:06:31   2024-07-25 11:36:51   1
  14   25      2024-07-25 12:18:17   2024-07-25 12:35:47   1
  15   15      2024-07-25 13:29:18   2024-07-25 13:58:48   1
  16   28      2024-07-25 14:32:46   2024-07-25 14:50:46   1
  17   14      2024-07-25 15:28:53   2024-07-25 15:52:53   1
  18   17      2024-07-25 16:51:33   2024-07-25 17:24:23   1
  19   Depot   2024-07-25 19:15:28   2024-07-25 19:15:28   1
  20   Depot   2024-07-25 07:00:00   2024-07-25 07:00:00   2
  21   3       2024-07-25 08:39:26   2024-07-25 08:55:06   2
  22   12      2024-07-25 09:33:57   2024-07-25 09:49:27   2
  23   7       2024-07-25 11:04:31   2024-07-25 11:19:31   2
  24   8       2024-07-25 12:09:06   2024-07-25 12:39:06   2
  25   1       2024-07-25 12:47:54   2024-07-25 13:06:44   2
  26   27      2024-07-25 14:15:27   2024-07-25 14:19:37   2
  27   21      2024-07-25 15:23:49   2024-07-25 15:30:29   2
  28   5       2024-07-25 16:15:04   2024-07-25 16:44:14   2
  29   9       2024-07-25 17:28:00   2024-07-25 17:31:20   2
  30   Depot   2024-07-25 19:20:09   2024-07-25 19:20:09   2

</div>
:::
:::
:::
:::
:::

::: {.jp-Cell .jp-MarkdownCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedMarkdown .jp-MarkdownOutput mime-type="text/markdown"}
The data shows the stations visited by each car arrival time and
departure time. Next I added the routes information to the map.
:::
:::
:::
:::

::: {.jp-Cell .jp-CodeCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
In \[ \]:
:::

::: {.jp-CodeMirrorEditor .jp-Editor .jp-InputArea-editor data-type="inline"}
::: {.cm-editor .cm-s-jupyter}
::: {.highlight .hl-ipython3}
    df.columns
:::
:::
:::
:::
:::

::: {.jp-Cell-outputWrapper}
::: {.jp-Collapser .jp-OutputCollapser .jp-Cell-outputCollapser}
:::

::: {.jp-OutputArea .jp-Cell-outputArea}
::: {.jp-OutputArea-child .jp-OutputArea-executeResult}
::: {.jp-OutputPrompt .jp-OutputArea-prompt}
Out\[ \]:
:::

::: {.jp-RenderedText .jp-OutputArea-output .jp-OutputArea-executeResult mime-type="text/plain" tabindex="0"}
    Index(['index', 'fas_id', 'name', 'address', 'postcode', 'easting', 'northing',
           'latitude', 'longitude', 'local_authority', 'Title', 'Amount',
           'Service', 'Service_hours', 'Open_Time', 'Close_Time'],
          dtype='object')
:::
:::
:::
:::
:::

::: {.jp-Cell .jp-CodeCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
In \[ \]:
:::

::: {.jp-CodeMirrorEditor .jp-Editor .jp-InputArea-editor data-type="inline"}
::: {.cm-editor .cm-s-jupyter}
::: {.highlight .hl-ipython3}
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
:::
:::
:::
:::
:::

::: {.jp-Cell-outputWrapper}
::: {.jp-Collapser .jp-OutputCollapser .jp-Cell-outputCollapser}
:::

::: {.jp-OutputArea .jp-Cell-outputArea}
::: {.jp-OutputArea-child .jp-OutputArea-executeResult}
::: {.jp-OutputPrompt .jp-OutputArea-prompt}
Out\[ \]:
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedHTML .jp-OutputArea-output .jp-OutputArea-executeResult mime-type="text/html" tabindex="0"}
::: {style="width:100%;"}
::: {style="position:relative;width:100%;height:0;padding-bottom:60%;"}
[Make this Notebook Trusted to load map: File -\> Trust
Notebook]{style="color:#565656"}
:::
:::
:::
:::
:::
:::
:::

::: {.jp-Cell .jp-MarkdownCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedMarkdown .jp-MarkdownOutput mime-type="text/markdown"}
References[¶](#References){.anchor-link} {#References}
----------------------------------------

1.  [VROOM Project API
    Documentation](https://github.com/VROOM-Project/vroom/blob/master/docs/API.md)
2.  [Vehicle Route Optimization in Python with
    OpenRouteService](https://syntaxbytetutorials.com/vehicle-route-optimization-in-python-with-openrouteservice/)
3.  [OpenRouteService](https://openrouteservice.org/)
4.  [Folium GeoJSON Marker
    Guide](https://python-visualization.github.io/folium/latest/user_guide/geojson/geojson_marker.html)
5.  [OpenRouteService Python Routing Optimization
    Example](https://github.com/GIScience/openrouteservice-examples/blob/master/python/Routing_Optimization_Idai.ipynb)
:::
:::
:::
:::

::: {.jp-Cell .jp-MarkdownCell .jp-Notebook-cell}
::: {.jp-Cell-inputWrapper tabindex="0"}
::: {.jp-Collapser .jp-InputCollapser .jp-Cell-inputCollapser}
:::

::: {.jp-InputArea .jp-Cell-inputArea}
::: {.jp-InputPrompt .jp-InputArea-prompt}
:::

::: {.jp-RenderedHTMLCommon .jp-RenderedMarkdown .jp-MarkdownOutput mime-type="text/markdown"}
:::
:::
:::
:::
