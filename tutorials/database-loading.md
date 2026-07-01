# 📥 Database Loading

This tutorial will guide you in using the <mark style="background-color:yellow;">**AISdb**</mark> package to load AIS data into a database and perform queries. We will begin with **AISdb** installation and environment setup, then proceed to examples of querying the loaded data and creating simple visualizations.

## Install Requirements <a href="#id-1.-install-requirements" id="id-1.-install-requirements"></a>

Preparing a Python virtual environment for AISdb is a safe practice. It allows you to manage dependencies and prevent conflicts with other projects, ensuring a clean and isolated setup for your work with AISdb. Run these commands in your terminal based on the operating system you are using:

{% code title="Linux" lineNumbers="true" %}
```bash
python -m venv AISdb         # create a python virtual environment
source ./AISdb/bin/activate  # activate the virtual environment
pip install aisdb            # from https://pypi.org/project/aisdb/
```
{% endcode %}

{% code title="Windows" lineNumbers="true" %}
```sh
python -m venv AISdb         # create a virtual environment
./AISdb/Scripts/activate     # activate the virtual environment
pip install aisdb            # install the AISdb package using pip
```
{% endcode %}

Now you can check your installation by running:

{% code lineNumbers="true" %}
```bash
$ python
>>> import aisdb
>>> aisdb.__version__        # should return '1.7.0' or newer
```
{% endcode %}

If you're using AISdb in [Jupyter](https://jupyter.org/) Notebook,  please include the following commands in your notebook cells:

{% code lineNumbers="true" %}
```bash
# install nest-asyncio for enabling asyncio.run() in Jupyter Notebook
%pip install nest-asyncio

# Some of the systems may show the following error when running the user interface:
# urllib3 v2.0 only supports OpenSSL 1.1.1+; currently, the 'SSL' module is compiled with 'LibreSSL 2.8.3'.
# install urllib3 v1.26.6 to avoid this error
%pip install urllib3==1.26.6
```
{% endcode %}

Then, import the required packages:

{% code lineNumbers="true" %}
```python
from datetime import datetime, timedelta
import os
import aisdb
import nest_asyncio
nest_asyncio.apply()
```
{% endcode %}

## Load AIS data into a database <a href="#id-2.-load-ais-data-into-a-database" id="id-2.-load-ais-data-into-a-database"></a>

This section will show you how to efficiently load AIS data into a database.&#x20;

AISdb includes two database connection approaches:&#x20;

1. SQLite database connection; and,
2. PostgreSQL database connection.

### SQLite database connection

We are working with the SQLite database in most of the usage scenarios. Here is an example of loading data using the sample data included in the AISdb package:

<pre class="language-python" data-line-numbers><code class="lang-python"># List the test data files included in the package
print(os.listdir(os.path.join(aisdb.sqlpath, '..', 'tests', 'testdata')))
# You will see the print result: 
# ['test_data_20210701.csv', 'test_data_20211101.nm4', 'test_data_20211101.nm4.gz']

# Set the path for the SQLite database file to be used
<a data-footnote-ref href="#user-content-fn-1">dbpath = './</a>test_database<a data-footnote-ref href="#user-content-fn-1">.db'</a>

# Use test_data_20210701.csv as the test data
filepaths = [os.path.join(aisdb.sqlpath, '..', 'tests', 'testdata', 'test_data_20210701.csv')]
with aisdb.DBConn(dbpath = dbpath) as dbconn:
    aisdb.decode_msgs(filepaths=filepaths, dbconn=dbconn, source='TESTING')
</code></pre>

The code above decodes the AIS messages from the CSV file specified in `filepaths` and inserts them into the SQLite database connected via `dbconn`.&#x20;

Following is a quick example of a **query** and **visualization** of the data we just loaded with AISdb:

{% code lineNumbers="true" %}
```python
start_time = datetime.strptime("2021-07-01 00:00:00", '%Y-%m-%d %H:%M:%S')
end_time = datetime.strptime("2021-07-02 00:00:00", '%Y-%m-%d %H:%M:%S')

with aisdb.SQLiteDBConn(dbpath=dbpath) as dbconn:
    qry = aisdb.DBQuery(
        dbconn=dbconn,
        dbpath='./AIS2.db',
        callback=aisdb.database.sql_query_strings.in_timerange,
        start=start_time,
        end=end_time,
    )
    rowgen = qry.gen_qry()
    tracks = aisdb.track_gen.TrackGen(rowgen, decimate=False)

    if __name__ == '__main__':
        aisdb.web_interface.visualize(
            tracks,
            visualearth=True,
            open_browser=True,
        )
```
{% endcode %}

<figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption><p>Visualization of vessel tracks queried from SQLite database created from test data</p></figcaption></figure>

### PostgreSQL database connection

In addition to SQLite database connection, PostgreSQL is used in AISdb for its superior concurrency handling and data-sharing capabilities, making it suitable for collaborative environments and handling larger datasets efficiently. The structure and interactions with PostgreSQL are designed to provide robust and scalable solutions for AIS data storage and querying. For PostgreSQL, you need the `psycopg2` library:

{% code lineNumbers="true" %}
```bash
pip install psycopg2
```
{% endcode %}

To connect to a PostgreSQL database, AISdb uses the `PostgresDBConn` class:

{% code lineNumbers="true" %}
```python
from aisdb.database.dbconn import PostgresDBConn

# Option 1: Using keyword arguments
dbconn = PostgresDBConn(
    hostaddr='127.0.0.1',      # Replace with the PostgreSQL address
    port=5432,                 # Replace with the PostgreSQL running port
    user='USERNAME',           # Replace with the PostgreSQL username
    password='PASSWORD',       # Replace with your password
    dbname='aisviz'            # Replace with your database name
)

# Option 2: Using a connection string
dbconn = PostgresDBConn('postgresql://USERNAME:PASSWORD@HOST:PORT/DATABASE')
```
{% endcode %}

After establishing a connection to the PostgreSQL database, specifying the path of the data files, and using the `aisdb.decode_msgs` function for data processing, the following operations will be performed in order: data files processing, table creation, data insertion, and index rebuilding.

Please pay close attention to the flags in `aisdb.decode_msgs`, as recent updates provide more flexibility for database configurations. These updates include support for ingesting NOAA data into the `aisdb` format and the option to structure tables using either the original **B-Tree indexes** or **TimescaleDB’s structure** when the extension is enabled. In particular, please take care of the following parameters:

* **`source`** _(str, optional)_\
  Specifies the data source to be processed and loaded into the database.
  * Options: `"Spire"`, `"NOAA"`/`"noaa"`, or leave empty.
  * **Default**: empty but will progress with Spire source.
* **`raw_insertion`** _(bool, optional)_
  * If `False`, the function will drop and rebuild indexes to **speed up data loading**.
  * **Default**: `True`.
* **`timescaledb`** _(bool, optional)_
  * Set to `True` **only if** using the TimescaleDB extension in your PostgreSQL database.
  * Refer to the [TimescaleDB documentation](https://docs.timescale.com/self-hosted/latest/) for proper setup and usage.

### Example: Processing a Full Year of Spire Data (2024)

The following example demonstrates how to process and load Spire data for the entire year 2024 into an `aisdb` database with the TimescaleDB extension installed:

```python
start_year = 2024
end_year = 2024
start_month = 1
end_month = 12

overall_start_time = time.time()

for year in range(start_year, end_year + 1):
    for month in range(start_month, end_month + 1):
        print(f'Loading {year}{month:02d}')
        month_start_time = time.time()

        filepaths = aisdb.glob_files(f'/slow-array/Spire/{year}{month:02d}/','.zip')
        filepaths = sorted([f for f in filepaths if f'{year}{month:02d}' in f])
        print(f'Number of files: {len(filepaths)}')

        with aisdb.PostgresDBConn(libpq_connstring=psql_conn_string) as dbconn:
            try:
                aisdb.decode_msgs(filepaths,
                                dbconn=dbconn,
                                source='Spire',
                                verbose=True,
                                skip_checksum=True,
                                raw_insertion=True,
                                workers=6,
                                timescaledb=True,
                        )
            except Exception as e:
                print(f'Error loading {year}{month:02d}: {e}')
                continue
```

Example of performing queries and visualizations with PostgreSQL database:

{% code lineNumbers="true" %}
```python
from aisdb.gis import DomainFromPoints
from aisdb.database.dbqry import DBQuery
from datetime import datetime

# Define a spatial domain centered around the point (-63.6, 44.6) with a radial distance of 50000 meters.
domain = DomainFromPoints(points=[(-63.6, 44.6)], radial_distances=[50000])

# Create a query object to fetch AIS data within the specified time range and spatial domain.
qry = DBQuery(
    dbconn=dbconn,
    start=datetime(2023, 1, 1), end=datetime(2023, 2, 1),
    xmin=domain.boundary['xmin'], xmax=domain.boundary['xmax'],
    ymin=domain.boundary['ymin'], ymax=domain.boundary['ymax'],
    callback=aisdb.database.sqlfcn_callbacks.in_time_bbox_validmmsi
)

# Generate rows from the query
rowgen = qry.gen_qry()

# Convert the generated rows into tracks
tracks = aisdb.track_gen.TrackGen(rowgen, decimate=False)

# Visualize the tracks on a map
aisdb.web_interface.visualize(
    tracks,           # The tracks (trajectories) to visualize.
    domain=domain,    # The spatial domain to use for the visualization.
    visualearth=True, # If True, use Visual Earth for the map background.
    open_browser=True # If True, automatically open the visualization in a web browser.
)
```
{% endcode %}

<figure><img src="../.gitbook/assets/Screenshot from 2024-09-04 11-09-25.png" alt=""><figcaption><p>Visualization of tracks queried from PostgreSQL database</p></figcaption></figure>

Moreover, if you wish to use your own AIS data to create and process a database with AISdb, please check out our instructional guide on data processing and database creation: [_Using Your AIS Data_](using-your-ais-data.md)_._

[^1]: 
