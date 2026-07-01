# 🌊 Bathymetric Data

## Process AIS data with Bathymetric Data <a href="#id-5.-process-ais-data-with-external-data-source" id="id-5.-process-ais-data-with-external-data-source"></a>

This section demonstrates integrating AIS data with external bathymetric data to enrich our analysis. In the following example, we identified all vessels within a 500-kilometer radius around the central area of Halifax, Canada, on January 1, 2018.

### Raster file preparation

First, we imported the necessary packages and prepared the bathymetry data. It’s important to note that the downloaded bathymetric data is divided into eight segments, organized by latitude and longitude. In a later step, you will need to select the appropriate bathymetric raster file based on the geographical region covered by your vessel track data.

{% code lineNumbers="true" %}
```python
import os
import aisdb
import nest_asyncio

from datetime import datetime
from aisdb.database.dbconn import SQLiteDBConn
from aisdb import DBConn, DBQuery, DomainFromPoints

nest_asyncio.apply()

# set the path to the data storage directory
bathymetry_data_dir = "./bathymetry_data/"

# check if the directory exists
if not os.path.exists(bathymetry_data_dir):
    os.makedirs(bathymetry_data_dir)

# check if the directory is empty
if os.listdir(bathymetry_data_dir) == []:
    # download the bathymetry data
    bathy = aisdb.webdata.bathymetry.Gebco(data_dir=bathymetry_data_dir)
    bathy.fetch_bathymetry_grid()
else:
    print("Bathymetry data already exists.")
```
{% endcode %}

### Coloring the tracks

We defined a coloring criterion to classify tracks based on their average depths relative to the bathymetry. Tracks that traverse shallow waters with an average depth of less than 100 meters are colored in yellow. Those spanning depths between 100 and 1,000 meters are represented in orange, indicating a transition to deeper waters. As the depth increases, tracks reaching up to 20 kilometers are marked pink. The deepest tracks, descending beyond 20 kilometers, are distinctly colored in red.

{% code lineNumbers="true" %}
```python
def add_color(tracks):
    for track in tracks:
        # Calculate the average coastal distance
        avg_coast_distance = sum(abs(dist) for dist in track['coast_distance']) / len(track['coast_distance'])
        
        # Determine the color based on the average coastal distance
        if avg_coast_distance <= 100:
            track['color'] = "yellow"
        elif avg_coast_distance <= 1000:
            track['color'] = "orange"
        elif avg_coast_distance <= 20000:
            track['color'] = "pink"
        else:
            track['color'] = "red"
        
        yield track
```
{% endcode %}

### Integration with the bathymetric raster file

Next, we query the AIS data to be integrated with the bathymetric raster file and apply the coloring function to mark the tracks based on their average depths relative to the bathymetry.

{% code lineNumbers="true" %}
```python
dbpath = 'YOUR_DATABASE.db' # define the path to your database
end_time = datetime.strptime("2018-01-02 00:00:00", '%Y-%m-%d %H:%M:%S')
start_time = datetime.strptime("2018-01-01 00:00:00", '%Y-%m-%d %H:%M:%S')
domain = DomainFromPoints(points=[(-63.6, 44.6)], radial_distances=[500000])

with SQLiteDBConn(dbpath=dbpath) as dbconn:
    qry = DBQuery(
        dbconn=dbconn, start=start_time, end=end_time,
        xmin=domain.boundary['xmin'], xmax=domain.boundary['xmax'],
        ymin=domain.boundary['ymin'], ymax=domain.boundary['ymax'],
        callback=aisdb.database.sqlfcn_callbacks.in_time_bbox_validmmsi,
    )
    tracks = aisdb.track_gen.TrackGen(qry.gen_qry(), decimate=False)

    # Merge the tracks with the raster data
    raster_path = "./bathymetry_data/gebco_2022_n90.0_s0.0_w-90.0_e0.0.tif"
    raster = aisdb.webdata.load_raster.RasterFile(raster_path)
    tracks_raster = raster.merge_tracks(tracks, new_track_key="coast_distance")

    # Add color to the tracks
    tracks_colored = add_color(tracks_raster)

    if __name__ == '__main__':
        aisdb.web_interface.visualize(
            tracks_colored,
            visualearth=True,
            open_browser=True,
        )
```
{% endcode %}

The integrated results are color-coded and can be visualized as shown below:

<figure><img src="../.gitbook/assets/image (41).png" alt=""><figcaption><p>Vessel tracks colored with average depths relative to the bathymetry</p></figcaption></figure>

&#x20;Example of using bathymetry data to color-code vessel tracks based on their average depth:

* **Yellow:** Tracks with an average depth of less than 100 meters (shallow waters).
* **Orange:** Tracks with an average depth between 100 and 1,000 meters (transition to deeper waters).
* **Pink:** Tracks with an average depth between 1,000 and 20,000 meters (deeper waters).
* **Red:** Tracks with an average depth greater than 20,000 meters (deepest waters).
