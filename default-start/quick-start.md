---
description: A hands-on quick start guide for using AISdb.
cover: >-
  https://images.unsplash.com/photo-1505896121-d0448a07f27c?crop=entropy&cs=srgb&fm=jpg&ixid=M3wxOTcwMjR8MHwxfHNlYXJjaHw1fHxOb3ZhJTIwU2NvdGlhfGVufDB8fHx8MTcyMzMxMTE3Mnww&ixlib=rb-4.0.3&q=85
coverY: 0
---

# 🛰️ Quick Start

### If you are new to AIS topics, [click-here](https://aisviz.gitbook.io/documentation/tutorials/ais-automatic-identification-system) to know about "Automatic Identification System (AIS)".&#x20;

**Note:** _If you are starting from scratch, download the data ".db" file in our_  [_AISdb Tutorial GitHub_](https://github.com/AISViz/Tutorials) _repository so that you can follow this guide properly._

### Python Environment and Installation

To work with the AISdb Python package, please ensure you have Python version 3.8 or higher. If you plan to use SQLite, no additional installation is required, as it is included with Python by default. However, those who prefer using a PostgreSQL server must install it separately and enable the TimescaleDB extension to function correctly.

#### User Installation

The AISdb Python package can be conveniently installed using pip. <mark style="background-color:yellow;">It's highly recommended that a virtual Python environment be created and the package installed within it.</mark>

<pre class="language-bash" data-title="Linux" data-line-numbers><code class="lang-bash"><strong>python -m venv AISdb   # create a python virtual environment
</strong>source ./AISdb/bin/activate  # activate the virtual environment
<strong>pip install aisdb  # from https://pypi.org/project/aisdb/
</strong></code></pre>

<pre class="language-bash" data-title="Windows" data-line-numbers><code class="lang-bash"><strong>python -m venv AISdb
</strong>./AISdb/Scripts/activate  
<strong>pip install aisdb
</strong></code></pre>

You can test your installation by running the following commands:

<pre class="language-python" data-line-numbers><code class="lang-python"><strong>python
</strong>>>> import aisdb
<strong>>>> aisdb.__version__  # should return '1.7.3' or newer
</strong></code></pre>

Notice that if you are running [Jupyter](https://jupyter.org/), ensure it is installed in the same environment as AISdb:

<pre class="language-bash" data-line-numbers><code class="lang-bash"><strong>source ./AISdb/bin/activate
</strong>pip install jupyter
<strong>jupyter notebook
</strong></code></pre>

The Python code in the rest of this document can be run in the Python environment you created.&#x20;

#### Development Installation

For using _<mark style="background-color:red;">nightly builds</mark>_ <mark style="background-color:red;"></mark><mark style="background-color:red;">**(not mandatory)**</mark>, you can install it from the source:

<pre class="language-bash" data-line-numbers><code class="lang-bash"><strong>source AISdb/bin/activate  # On Windows use `AISdb\Scripts\activate`
</strong>
<strong># Cloning the Repository and installing the package
</strong>git clone https://github.com/AISViz/AISdb.git &#x26;&#x26; cd aisdb
<strong>
</strong># Windows users can instead download the installer:
<strong>#   - https://forge.rust-lang.org/infra/other-installation-methods.html#rustup
</strong>#   - https://static.rust-lang.org/rustup/dist/i686-pc-windows-gnu/rustup-init.exe
<strong>curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs > install-rust.sh
</strong>
<strong># Installing Rust and Maturin
</strong>/bin/bash install-rust.sh -q -y
<strong>pip install --upgrade maturin[patchelf]
</strong>
<strong># Building AISdb package with Maturin
</strong>maturin develop --release --extras=test,docs
</code></pre>

Alternatively, you can use _<mark style="background-color:red;">nightly builds</mark>_ <mark style="background-color:red;"></mark><mark style="background-color:red;">**(not mandatory)**</mark> on **Google Colab** as follows:

<pre class="language-python" data-line-numbers><code class="lang-python"><strong>import os
</strong># Clone the AISdb repository from GitHub
<strong>!git clone https://github.com/AISViz/AISdb.git
</strong># Install Rust using the official Rustup script
<strong>!curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
</strong># Install Maturin to build the packages
<strong>!pip install --upgrade maturin[patchelf]
</strong># Set up environment variables
<strong>os.environ["PATH"] += os.pathsep + "/root/.cargo/bin"
</strong># Install wasm-pack for building WebAssembly packages
<strong>!curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh
</strong># Install wasm-pack as a Cargo dependency
<strong>!cargo install wasm-pack
</strong># Setting environment variable for the virtual environment
<strong>os.environ["VIRTUAL_ENV"] = "/usr/local"
</strong># Change directory to AISdb for building the package
<strong>%cd AISdb
</strong># Build and install the AISdb package using Maturin
<strong>!maturin develop --release --extras=test,docs
</strong></code></pre>

### Database Handling

AISdb supports SQLite and PostgreSQL databases. <mark style="background-color:red;">Since version</mark> <mark style="background-color:red;"></mark><mark style="background-color:red;">**1.7.3**</mark><mark style="background-color:red;">, AISdb requires</mark> [<mark style="background-color:red;">**TimescaleDB over PostgreSQL**</mark>](https://www.timescale.com/) <mark style="background-color:red;">to function properly.</mark> To install TimescaleDB, follow these steps:

_Install TimescaleDB (PostgreSQL Extension)_

```bash
$ sudo apt install -y timescaledb-postgresql-XX  # XX is the PG-SQL version
```

_Enable the Extension in PostgreSQL_

```sql
> CREATE EXTENSION IF NOT EXISTS timescaledb;
```

_Verify the Installation_

```sql
> SELECT * FROM timescaledb_information.version;
```

_Restart PostgreSQL_

```bash
$ sudo systemctl restart postgresql
```

#### Connecting to a **PostgreSQL** database

This option requires an optional dependency `psycopg` for interfacing with Postgres databases. Beware that Postgres accepts these [keyword arguments](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-PARAMKEYWORDS)[.](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-PARAMKEYWORDS) Alternatively, a connection string may be used. Information on connection strings and Postgres URI format can be found [here](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING).

<pre class="language-python" data-line-numbers><code class="lang-python"><strong>from aisdb.database.dbconn import PostgresDBConn
</strong>
<strong># [OPTION 1]
</strong>dbconn = PostgresDBConn(
<strong>    hostaddr='127.0.0.1',  # Replace this with the Postgres address (supports IPv6)
</strong>    port=5432,  # Replace this with the Postgres running port (if not the default)
<strong>    user='USERNAME',  # Replace this with the Postgres username
</strong>    password='PASSWORD',  # Replace this with your password
<strong>    dbname='DATABASE',  # Replace this with your database name
</strong>)
<strong>
</strong># [OPTION 2]
<strong>dbconn = PostgresDBConn('postgresql://USERNAME:PASSWORD@HOST:PORT/DATABASE')
</strong></code></pre>

#### Attaching a SQLite database to AISdb

Querying SQLite is as easy as informing the name of a <mark style="background-color:red;">".db" file</mark> with the same entity-relationship as the databases supported by AIS, which are detailed in the [SQL Database](sql-database.md) section. We prepared an example SQLite database `example_data.db` based AIS data in a small region near Maine, United States in Jan 2022 from [Marine Cadastre](https://hub.marinecadastre.gov/pages/vesseltraffic), which is available in AISdb [Tutorial](https://github.com/AISViz/Tutorials) GitHub repository.

<pre class="language-python" data-line-numbers><code class="lang-python"><strong>from aisdb.database.dbconn import SQLiteDBConn 
</strong>
<strong>dbpath='example_data.db'
</strong>dbconn = SQLiteDBConn(dbpath=dbpath)
</code></pre>

If you want to create <mark style="background-color:yellow;">your database using your data</mark>, we have a [tutorial](../tutorials/using-your-ais-data.md) with examples that show you how to create an SQLite database from open-source data.&#x20;

### Querying the Database

Parameters for the database query can be defined using [`aisdb.database.dbqry.DBQuery`](https://aisviz.cs.dal.ca/AISdb/api/aisdb.database.dbqry.html#module-aisdb.database.dbqry). Iterate over rows returned from the database for each vessel with [`aisdb.database.dbqry.DBQuery.gen_qry()`](https://aisviz.cs.dal.ca/AISdb/api/aisdb.database.dbqry.html#module-aisdb.database.dbqry). Convert the results into generator-yielding dictionaries with NumPy arrays describing position vectors, _e.g._, lon, lat, and time, using [`aisdb.track_gen.TrackGen()`](https://aisviz.cs.dal.ca/AISdb/api/aisdb.track_gen.html#aisdb.track_gen.TrackGen).

The following query will return vessel trajectories from a given 1-hour time window:

<pre class="language-python" data-line-numbers><code class="lang-python"><strong>import aisdb
</strong>import pandas as pd
<strong>from datetime import datetime
</strong>from collections import defaultdict
<strong>
</strong>dbpath = 'example_data.db'
<strong>start_time = datetime.strptime("2022-01-01 00:00:00", '%Y-%m-%d %H:%M:%S')
</strong>end_time = datetime.strptime("2022-01-01 0:59:59", '%Y-%m-%d %H:%M:%S')
<strong>
</strong>def data2frame(tracks):
<strong>    # Dictionary where for key/value
</strong>    ais_data = defaultdict(lambda: pd.DataFrame(
        columns = ['time', 'lat', 'lon', 'cog', 'rocog', 'sog', 'delta_sog']))
<strong>
</strong>    for track in tracks:
<strong>        mmsi = track['mmsi']
</strong>        df = pd.DataFrame({
<strong>            'time': pd.to_datetime(track['time'], unit='s'),
</strong>            'lat': track['lat'], 'lon': track['lon'],
<strong>            'cog': track['cog'], 'sog': track['sog']
</strong>        })
<strong>
</strong>        # Sort by time in descending order
<strong>        df = df.sort_values(by='time', ascending=False).reset_index(drop=True)
</strong>        # Compute the time difference in seconds
<strong>        df['time_diff'] = df['time'].diff().dt.total_seconds()
</strong>        # Compute RoCOG (Rate of Change of Course Over Ground)
<strong>        delta_cog = (df['cog'].diff() + 180) % 360 - 180
</strong>        df['rocog'] = delta_cog / df['time_diff']
<strong>        # Compute Delta SOG (Rate of Change of Speed Over Ground)
</strong>        df['delta_sog'] = df['sog'].diff() / df['time_diff']
<strong>        # Fill NaN values (first row) and infinite values (division by zero cases)
</strong>        df[['rocog', 'delta_sog']] = df[['rocog', 'delta_sog']].replace([float('inf'), float('-inf')], 0).fillna(0)
<strong>        # Drop unnecessary column
</strong>        df.drop(columns = ['time_diff'], inplace=True)
<strong>        # Store in the dictionary
</strong>        ais_data[mmsi] = df
<strong>    return ais_data
</strong>
with aisdb.SQLiteDBConn(dbpath=dbpath) as dbconn:
    qry = aisdb.DBQuery(
<strong>        dbconn=dbconn, start=start_time, end=end_time,
</strong>        callback=aisdb.database.sqlfcn_callbacks.in_timerange_validmmsi,
<strong>    )
</strong>
<strong>    rowgen = qry.gen_qry()
</strong>    tracks = aisdb.track_gen.TrackGen(rowgen, decimate=False)
<strong>    ais_data = data2frame(tracks)  # re-use previous function
</strong>
<strong># Display DataFrames
</strong>for key in ais_data.keys():
<strong>    print(ais_data[key])
</strong></code></pre>

A specific region can be queried for AIS data using [`aisdb.gis.Domain`](https://aisviz.cs.dal.ca/AISdb/api/aisdb.track_gen.html#aisdb.track_gen.TrackGen) or one of its sub-classes to define a collection of `shapely` polygon features. For this example, the domain contains a single bounding box polygon derived from a longitude/latitude coordinate pair and radial distance specified in meters. If multiple features are included in the domain object, the domain boundaries will encompass the convex hull of all features.

<pre class="language-python" data-line-numbers><code class="lang-python"><strong># a circle with a 100km radius around the location point
</strong>domain = aisdb.DomainFromPoints(points=[(-69.34, 41.55)], radial_distances=[100000])
<strong>
</strong>with aisdb.SQLiteDBConn(dbpath=dbpath) as dbconn:
<strong>    qry = aisdb.DBQuery(
</strong>        dbconn=dbconn, start=start_time, end=end_time,
<strong>        xmin=domain.boundary['xmin'], xmax=domain.boundary['xmax'],
</strong>        ymin=domain.boundary['ymin'], ymax=domain.boundary['ymax'],
<strong>        callback=aisdb.database.sqlfcn_callbacks.in_validmmsi_bbox,
</strong>    )
<strong>    rowgen = qry.gen_qry()
</strong>    tracks = aisdb.track_gen.TrackGen(rowgen, decimate=False)
<strong>    ais_data = data2frame(tracks)  # re-use previous function
</strong>
<strong># Display DataFrames
</strong>for key in ais_data.keys():
<strong>    print(ais_data[key])
</strong></code></pre>

Additional query callbacks for filtering by region, timeframe, identifier, etc. can be found in [`aisdb.database.sql_query_strings`](https://aisviz.cs.dal.ca/AISdb/api/aisdb.database.sql_query_strings.html) and [`aisdb.database.sqlfcn_callbacks`](https://aisviz.cs.dal.ca/AISdb/api/aisdb.database.sqlfcn_callbacks.html)`.`

### Processing

#### Voyage Modelling

The above generator can be input into a processing function, yielding modified results. For example, to model the activity of vessels on a per-voyage or per-transit basis, each voyage is defined as a continuous vector of positions where the time between observed timestamps never exceeds 24 hours.

<pre class="language-python" data-line-numbers><code class="lang-python"><strong>from datetime import timedelta
</strong>
<strong># Define a maximum time interval
</strong>maxdelta = timedelta(hours=24)
<strong>
</strong>with aisdb.SQLiteDBConn(dbpath=dbpath) as dbconn:
<strong>    qry = aisdb.DBQuery(
</strong>        dbconn=dbconn, start=start_time, end=end_time,
<strong>        xmin=domain.boundary['xmin'], xmax=domain.boundary['xmax'],
</strong>        ymin=domain.boundary['ymin'], ymax=domain.boundary['ymax'],
<strong>        callback=aisdb.database.sqlfcn_callbacks.in_validmmsi_bbox,
</strong>    )
<strong>    rowgen = qry.gen_qry()
</strong>    tracks = aisdb.track_gen.TrackGen(rowgen, decimate=False)
<strong>
</strong>    # Split the generated tracks into segments
<strong>    track_segments = aisdb.split_timedelta(tracks, maxdelta)
</strong>    ais_data = data2frame(track_segments)  # re-use previous function
<strong>    
</strong>    # Display DataFrames
<strong>    for key in ais_data.keys():
</strong>        print(ais_data[key])
</code></pre>

#### Data cleaning and MMSI deduplication

A common problem with <mark style="background-color:purple;">**AIS data is noise**</mark>, where multiple vessels might broadcast using the same identifier (sometimes simultaneously). In such cases, AISdb can denoise the data:

**(1) Denoising with Encoder:** The [`aisdb.denoising_encoder.encode_greatcircledistance()`](https://aisviz.cs.dal.ca/AISdb/api/aisdb.denoising_encoder.html#aisdb.denoising_encoder.encode_greatcircledistance) function checks the approximate distance between each vessel’s position. It separates vectors where a vessel couldn’t reasonably travel using the most direct path, such as speeds over 50 knots.

**(2) Distance and Speed Thresholds:** A distance and speed threshold limits the maximum distance or time between messages that can be considered continuous.

**(3) Scoring and Segment Concatenation:** A score is computed for each position delta, with sequential messages nearby at shorter intervals given a higher score. This score is calculated by dividing the Haversine distance by elapsed time. Any deltas with a score not reaching the minimum threshold are considered the start of a new segment. New segments are compared to the end of existing segments with the same vessel identifier; if the score exceeds the minimum, they are concatenated. If multiple segments meet the minimum score, the new segment is concatenated to the existing segment with the highest score.

Notice that processing functions may be executed in sequence as a chain or pipeline, so after segmenting the individual voyages as shown above, results can be input into the encoder to remove noise and correct for vessels with duplicate identifiers.

<pre class="language-python" data-line-numbers><code class="lang-python"><strong>distance_threshold = 20000  # the maximum allowed distance (meters) between consecutive AIS messages
</strong>speed_threshold = 50        # the maximum allowed vessel speed in consecutive AIS messages
<strong>minscore = 1e-6             # the minimum score threshold for track segment validation
</strong>
<strong>with aisdb.SQLiteDBConn(dbpath=dbpath) as dbconn:
</strong>    qry = aisdb.DBQuery(
<strong>        dbconn=dbconn, start=start_time, end=end_time,
</strong>        callback=aisdb.database.sqlfcn_callbacks.in_timerange,
<strong>    )
</strong>    rowgen = qry.gen_qry()
<strong>    tracks = aisdb.track_gen.TrackGen(rowgen, decimate=False)
</strong>    
<strong>    # Encode the track segments to clean and validate the track data
</strong>    tracks_encoded = aisdb.encode_greatcircledistance(tracks, 
<strong>                                                      distance_threshold=distance_threshold, 
</strong>                                                      speed_threshold=speed_threshold, 
<strong>                                                      minscore=minscore)
</strong>    ais_data = data2frame(tracks_encoded)  # re-use previous function
<strong>    
</strong>    # Display DataFrames
<strong>    for key in ais_data.keys():
</strong>        print(ais_data[key])
</code></pre>

#### Interpolating, geofencing, and filtering

Building on the above processing pipeline, the resulting cleaned trajectories can be geofenced and filtered for results contained by at least one domain polygon and interpolated for uniformity.

<pre class="language-python" data-line-numbers><code class="lang-python"><strong># Define a domain with a central point and corresponding radial distances
</strong>domain = aisdb.DomainFromPoints(points=[(-69.34, 41.55),], radial_distances=[100000,])
<strong>
</strong># Filter the encoded tracks to include only those within the specified domain
<strong>tracks_filtered = aisdb.track_gen.zone_mask(tracks_encoded, domain)
</strong>
<strong># Interpolate the filtered tracks with a specified time interval
</strong>tracks_interp = aisdb.interp_time(tracks_filtered, step=timedelta(minutes=15))
</code></pre>

Additional processing functions can be found in the [`aisdb.track_gen`](https://aisviz.cs.dal.ca/AISdb/api/aisdb.track_gen.html) module.

#### Exporting as CSV

The resulting processed voyage data can be exported in CSV format instead of being printed:

<pre class="language-python" data-line-numbers><code class="lang-python"><strong>aisdb.write_csv(tracks_interp, 'ais_processed.csv')
</strong></code></pre>

### Integration with external metadata

AISDB supports integrating external data sources such as bathymetric charts and other raster grids.

#### Bathymetric charts

To determine the approximate ocean depth at each vessel position, the[`aisdb.webdata.bathymetry`](https://aisviz.cs.dal.ca/AISdb/api/aisdb.webdata.bathymetry.html) module can be used.

<pre class="language-python" data-line-numbers><code class="lang-python"><strong>import aisdb
</strong>
<strong># Set the data storage directory
</strong>data_dir = './testdata/'
<strong>
</strong># Download bathymetry grid from the internet
<strong>bathy = aisdb.webdata.bathymetry.Gebco(data_dir=data_dir)
</strong>bathy.fetch_bathymetry_grid()
</code></pre>

Once the data has been downloaded, the [`Gebco()`](https://aisviz.cs.dal.ca/AISdb/api/aisdb.webdata.bathymetry.html) class may be used to append bathymetric data to tracks in the context of a [`TrackGen()`](https://aisviz.cs.dal.ca/AISdb/api/aisdb.track_gen.html#aisdb.track_gen.TrackGen) processing pipeline like the processing functions described above.

<pre class="language-python" data-line-numbers><code class="lang-python"><strong>tracks = aisdb.TrackGen(qry.gen_qry(), decimate=False)
</strong>tracks_bathymetry = bathy.merge_tracks(tracks) # merge tracks with bathymetry data
</code></pre>

Also, see [`aisdb.webdata.shore_dist.ShoreDist`](https://aisviz.cs.dal.ca/AISdb/api/aisdb.webdata.shore_dist.html) for determining the approximate nearest distance to shore from vessel positions.

#### Rasters

Similarly, arbitrary raster coordinate-gridded data may be appended to vessel tracks

<pre class="language-python" data-line-numbers><code class="lang-python"><strong>tracks = aisdb.TrackGen(qry.gen_qry())
</strong>raster_path './GMT_intermediate_coast_distance_01d.tif'
<strong>
</strong># Load the raster file
<strong>raster = aisdb.webdata.load_raster.RasterFile(raster_path)
</strong>
<strong># Merge the generated tracks with the raster data
</strong>tracks = raster.merge_tracks(tracks, new_track_key="coast_distance")
</code></pre>

### Visualization

AIS data from the database may be overlayed on a map such as the one shown above using the [`aisdb.web_interface.visualize()`](https://aisviz.cs.dal.ca/AISdb/api/aisdb.web_interface.html#aisdb.web_interface.visualize) function. This function accepts a generator of track dictionaries such as those output by [`aisdb.track_gen.TrackGen()`](https://aisviz.cs.dal.ca/AISdb/api/aisdb.track_gen.html#aisdb.track_gen.TrackGen).&#x20;

<pre class="language-python" data-line-numbers><code class="lang-python"><strong>from datetime import datetime, timedelta
</strong>import aisdb
<strong>from aisdb import DomainFromPoints
</strong>
<strong>dbpath='example_data.db'
</strong>
<strong>def color_tracks(tracks):
</strong>    ''' set the color of each vessel track using a color name or RGB value '''
<strong>    for track in tracks:
</strong>        track['color'] = 'blue' or 'rgb(0,0,255)'
<strong>        yield track
</strong>
<strong># Set the start and end times for the query
</strong>start_time = datetime.strptime("2022-01-01 00:00:00", '%Y-%m-%d %H:%M:%S')
<strong>end_time = datetime.strptime("2022-01-31 00:00:00", '%Y-%m-%d %H:%M:%S')
</strong>
<strong>with aisdb.SQLiteDBConn(dbpath=dbpath) as dbconn:
</strong>    qry = aisdb.DBQuery(
<strong>        dbconn=dbconn,
</strong>        start=start_time,
<strong>        end=end_time,
</strong>        callback=aisdb.database.sqlfcn_callbacks.in_timerange_validmmsi,
<strong>    )
</strong>    rowgen = qry.gen_qry()
<strong>    
</strong>    # Convert queried rows to vessel trajectories
<strong>    tracks = aisdb.track_gen.TrackGen(rowgen, decimate=False)
</strong>    
<strong>    # Visualization
</strong>    aisdb.web_interface.visualize(
<strong>        tracks,
</strong>        visualearth=False,
<strong>        open_browser=True,
</strong>    )
</code></pre>

<figure><img src="../.gitbook/assets/image (5).png" alt=""><figcaption><p>Visualization of vessel tracks within a defined time range</p></figcaption></figure>

For a complete plug-and-play solution, you may clone our [Google Colab Notebook](https://colab.research.google.com/drive/1nfDUNfw7WSa5FuxRiggbK2Rvyc5qL5iH?usp=sharing).
