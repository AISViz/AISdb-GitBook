# 🌦️ Weather Data

This tutorial introduces integrating weather data from GRIB files with AIS data for enhanced vessel tracking analysis. Practical examples are provided below illustrating how to integrate AISdb tracks with the weather data in GRIB files. &#x20;

To directly work with the jupyter notebook, click here: [https://github.com/AISViz/AISdb/blob/master/examples/weather.ipynb](https://github.com/AISViz/AISdb/blob/master/examples/weather.ipynb)

## **Prerequisites**

Before diving in, users are expected to have the following set-up: a Copernicus CDS account (free) to access ERA5 data, which can be obtained through the [ECMWF-Signup](https://accounts.ecmwf.int/auth/realms/ecmwf/login-actions/registration?client_id=cds\&tab_id=sBIo0pbduZ8), and AISdb set up either locally or remotely. Refer to the [AISDB-Installation](https://aisviz.gitbook.io/documentation/default-start/quick-start#python-environment-and-installation) documentation for detailed instructions and configuration options.

## Usage:

Once you have a Copernicus CDS account and AISdb installed, you can **download** weather data in GRIB format directly from the CDS and use AISdb to **extract** specific variables from those files.

AISdb supports both **zipped (`.zip`)** and **uncompressed GRIB (`.grib`)** files. These files should be named using the `yyyy-mm` format (e.g., `2023-03.grib` or `2023-03.zip`) and placed in a folder such as `/home/CanadaV2`.

With the `WeatherDataStore` class in AISdb, you can specify:

* The desired **weather variables** (e.g., `'10u'`, `'10v'` ,`tp` ),
* A **date range** (e.g., August 1 to August 30, 2023),
* And the **directory** where the GRIB files are stored.

To **automatically download GRIB files** from the Copernicus Climate Data Store (CDS), AISdb provides a convenient option using the `WeatherDataStore` class.

Simply set the parameter `download_from_cds=True`, and specify the required **weather variable short names**, **date range**, **target area**, and **output path**.

To extract weather data for specific latitude, longitude, and timestamp values, call the `yield_tracks_with_weather()` method. This returns a dictionary containing the requested weather variables for each location-time pair.

```python
from aisdb.weather.data_store import WeatherDataStore # for weather

# ...some code before...
with aisdb.SQLiteDBConn(dbpath=dbpath) as dbconn:
    qry = aisdb.DBQuery(
        dbconn=dbconn,
        start=start_time,
        end=end_time,
        callback=aisdb.database.sqlfcn_callbacks.in_timerange_validmmsi,
    )
    rowgen = qry.gen_qry()
    
    # Convert queried rows to vessel trajectories
    tracks = aisdb.track_gen.TrackGen(rowgen, decimate=False)
    
    # Mention the short_names for the required weather data from the grib file
    weather_data_store = WeatherDataStore(short_names = ['10u','10v','tp'], start = start_time,end =  end_time,weather_data_path = ".",download_from_cds = False ,area = [-70, 45, -58, 53])

    tracks = weather_data_store.yield_tracks_with_weather(tracks)
    
    for t in tracks:
        print(f"'u-component' 10m wind for:\nlat: {t['lat'][0]} \nlon: {t['lon'][0]} \ntime: {t['time'][0]} \nis {t['weather_data']['u10'][0]} m/s")
        break
    
    weather_data_store.close()
```

Output:&#x20;

<pre class="language-bash"><code class="lang-bash"><strong>'u-component' 10m wind for: 
</strong>lat: 50.003334045410156 
lon: -66.76000213623047 
time: 1690858823 
is 1.9680767059326172 m/s
</code></pre>

## **What are `short_names`?**

In ECMWF (European Centre for Medium-Range Weather Forecasts) terminology, a "short name" is a concise, often abbreviated, identifier used to represent a specific meteorological parameter or variable within their data files (like GRIB files). It typically refers to a concise identifier used for climate and weather data variables. For example, "t2m" is a short name for "2-meter temperature".

For a list of short names for different weather components, refer to: [https://confluence.ecmwf.int/display/CKB/ERA5%3A+data+documentation#heading-Parameterlistings](https://confluence.ecmwf.int/display/CKB/ERA5%3A+data+documentation#heading-Parameterlistings)&#x20;

## **Complete Walkthrough Over an Example**

Let's work on an example where we retrieve AIS tracks from AISdb , call WeatherDataStore to add weather data to the tracks.

## **Step 1: Import all necessary packages**

```python
import aisdb
import nest_asyncio
from aisdb import DBQuery
from aisdb.database.dbconn import PostgresDBConn
from datetime import datetime
from PIL import ImageFile
from aisdb.weather.data_store import WeatherDataStore # for weather

nest_asyncio.apply()
ImageFile.LOAD_TRUNCATED_IMAGES = True
```

## **Step 2: Connect to the database**

```python
# >>> PostgreSQL Information <<<
db_user=''            # DB User
db_dbname='aisviz'         # DB Schema
db_password=''    # DB Password
db_hostaddr='127.0.0.1'    # DB Host address

dbconn = PostgresDBConn(
    port=5555,             # PostgreSQL port
    user=db_user,          # PostgreSQL username
    dbname=db_dbname,      # PostgreSQL database
    host=db_hostaddr,      # PostgreSQL address
    password=db_password,  # PostgreSQL password
)
```

## **Step 3: Query the required tracks**

Specify the region and duration for which you wish the tracks to be generated. The `TrackGen` returns a generator containing all the dynamic and static column values of AIS data.

```python
xmin, ymin, xmax, ymax = -70, 45, -58, 53
gulf_bbox = [xmin, xmax, ymin, ymax]
start_time = datetime(2023, 8, 1)
end_time = datetime(2023, 8, 30)

qry = DBQuery(
    dbconn=dbconn,
    start=start_time, end=end_time,
    xmin=xmin, xmax=xmax, ymin=ymin, ymax=ymax,
    callback=aisdb.database.sqlfcn_callbacks.in_time_bbox_validmmsi
)

ais_tracks = []
rowgen = qry.gen_qry()
tracks = aisdb.track_gen.TrackGen(rowgen, decimate=True")
```

## Step 4: Specify the necessary weather components using short\_name convention

Next, to merge the tracks with the weather data, we need to use the `WeatherDataStore` class from `aisdb.weather.era5`. By calling the `WeatherDataStore` with the required weather components (using their short names) and providing the path to your GRIB file containing the weather dataset, it will open the file and return an object through which you can perform further operations.

```python
weather_data_store = WeatherDataStore(
    short_names=['10u', '10v', 'tp'],       # U & V wind components, total precipitation
    start=start_time,                      # e.g., datetime(2023, 8, 1)
    end=end_time,                          # e.g., datetime(2023, 8, 30)
    weather_data_path=".",                 # Local folder to store downloaded GRIBs
    download_from_cds=True,                # Enable download from CDS
    area=[-70, 45, -58, 53]                # [west, north, east, south] in degrees
)
```

Here, `10v` and `10u` are the [10 metre U wind component](https://apps.ecmwf.int/codes/grib/param-db/165) and the [10 metre V wind component](https://apps.ecmwf.int/codes/grib/param-db/166).

## Step 5: Fetch Weather Values for a given latitude ,longiture and time

By using the method `weather_data_store.yield_tracks_with_weather(tracks)`, the tacks are concatenated with weather data.

Example usage:

```python
tracks_with_weather = weather_data_store.yield_tracks_with_weather(tracks)
    
for track in tracks_with_weather :
    print(f"'u-component' 10m wind for:\nlat: {track['lat'][0]} \nlon: {track['lon'][0]} \ntime: {track['time'][0]} \nis {track['weather_data']['u10'][0]} m/s")
    break
    
weather_data_store.close() # gracefully closes the opened GRIB file.
```

## Why do we need to integrate weather data with AIS data?

By integrating weather data with AIS data , we can study the patterns of ship movement in relation to weather conditions. This integration allows us to analyze how factors such as wind, sea surface temperature, and atmospheric pressure influence vessel trajectories. By merging dynamic AIS data with detailed climate variables, we can gain deeper insights into the environmental factors affecting shipping routes and optimize vessel navigation for better efficiency and safety.

<figure><img src="../.gitbook/assets/image (42).png" alt=""><figcaption><p>U-V 100m component wind over Gulf Of St. Lawrance for Aug 2018</p></figcaption></figure>
