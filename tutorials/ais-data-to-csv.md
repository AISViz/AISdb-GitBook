# 📒 AIS Data to CSV

Building on the previous section, where we used AIS data to create AISdb databases, users can export AIS data from these databases into CSV format. In this section, we provide examples of exporting data from <mark style="background-color:yellow;">SQLite or PostgreSQL databases into CSV files</mark>. While we demonstrate these operations using internal data, you can apply the same techniques to your databases.&#x20;

## Export CSV from SQLite Database

In the first example, we connected to a SQLite database, queried data in a specific time range and area of interest, and then exported the queried data to a CSV file:

{% code lineNumbers="true" %}
```python
import csv
import aisdb
import nest_asyncio

from aisdb import DBConn, DBQuery, DomainFromPoints
from aisdb.database.dbconn import SQLiteDBConn
from datetime import datetime

nest_asyncio.apply()

dbpath = 'YOUR_DATABASE.db' # Path to your database
end_time = datetime.strptime("2018-01-02 00:00:00", '%Y-%m-%d %H:%M:%S')
start_time = datetime.strptime("2018-01-01 00:00:00", '%Y-%m-%d %H:%M:%S')
domain = DomainFromPoints(points=[(-63.6, 44.6)], radial_distances=[50000])

# Connect to SQLite database
dbconn = SQLiteDBConn(dbpath=dbpath)

with SQLiteDBConn(dbpath=dbpath) as dbconn:
    qry = DBQuery(
        dbconn=dbconn, start=start_time, end=end_time,
        xmin=domain.boundary['xmin'], xmax=domain.boundary['xmax'],
        ymin=domain.boundary['ymin'], ymax=domain.boundary['ymax'],
        callback=aisdb.database.sqlfcn_callbacks.in_time_bbox_validmmsi,
    )
    tracks = aisdb.track_gen.TrackGen(qry.gen_qry(), decimate=False)

# Define the headers for the CSV file
headers = ['mmsi', 'time', 'lon', 'lat', 'cog', 'sog',
           'utc_second', 'heading', 'rot', 'maneuver']

# Open the CSV file for writing
csv_filename = 'output_sqlite.csv'
with open(csv_filename, mode='w', newline='') as file:
    writer = csv.DictWriter(file, fieldnames=headers)
    writer.writeheader()  # Write the header once
    
    for track in tracks:
        for i in range(len(track['time'])):
            row = {
                'rot': track['rot'],
                'mmsi': track['mmsi'],
                'lon': track['lon'][i],
                'lat': track['lat'][i],
                'cog': track['cog'][i],
                'sog': track['sog'][i],
                'time': track['time'][i],
                'heading': track['heading'],
                'maneuver': track['maneuver'],
                'utc_second': track['utc_second'][i],
            }
            writer.writerow(row)  # Write the row to the CSV file

print(f"All tracks have been combined and written to {csv_filename}")
```
{% endcode %}

Now we can check the data in the exported CSV file:

{% code lineNumbers="true" %}
```
mmsi 	time 	lon 	lat 	cog 	sog 	utc_second 	heading 	rot 	maneuver
0 	219014000 	1514767484 	-63.537167 	44.635834 	322 	0.0 	44 	295.0 	0.0 	0
1 	219014000 	1514814284 	-63.537167 	44.635834 	119 	0.0 	45 	295.0 	0.0 	0
2 	219014000 	1514829783 	-63.537167 	44.635834 	143 	0.0 	15 	295.0 	0.0 	0
3 	219014000 	1514829843 	-63.537167 	44.635834 	171 	0.0 	15 	295.0 	0.0 	0
4 	219014000 	1514830042 	-63.537167 	44.635834 	3 	0.0 	35 	295.0 	0.0 	0
```
{% endcode %}

## Export CSV from PostgreSQL Database

Similar to exporting data from a SQLite database to a CSV file, the only difference this time is that you'll need to connect to your PostgreSQL database and query the data you want to export to CSV. We showed a full example as follows:

{% code lineNumbers="true" %}
```python
import csv
import aisdb
import nest_asyncio

from datetime import datetime
from aisdb.database.dbconn import PostgresDBConn
from aisdb import DBConn, DBQuery, DomainFromPoints

nest_asyncio.apply()

dbconn = PostgresDBConn(
    host='localhost',          # PostgreSQL address
    port=5432,                 # PostgreSQL port
    user='your_username',      # PostgreSQL username
    password='your_password',  # PostgreSQL password
    dbname='database_name'     # Database name
)

qry = DBQuery(
    dbconn=dbconn,
    start=datetime(2023, 1, 1), end=datetime(2023, 1, 3),
    xmin=domain.boundary['xmin'], xmax=domain.boundary['xmax'],
    ymin=domain.boundary['ymin'], ymax=domain.boundary['ymax'],
    callback=aisdb.database.sqlfcn_callbacks.in_time_bbox_validmmsi
)

tracks = aisdb.track_gen.TrackGen(qry.gen_qry(), decimate=False)

# Define the headers for the CSV file
headers = ['mmsi', 'time', 'lon', 'lat', 'cog', 'sog',
           'utc_second', 'heading', 'rot', 'maneuver']

# Open the CSV file for writing
csv_filename = 'output_postgresql.csv'
with open(csv_filename, mode='w', newline='') as file:
    writer = csv.DictWriter(file, fieldnames=headers)
    writer.writeheader()  # Write the header once
    
    for track in tracks:
        for i in range(len(track['time'])):
            row = {
                'rot': track['rot'],
                'mmsi': track['mmsi'],
                'lon': track['lon'][i],
                'lat': track['lat'][i],
                'cog': track['cog'][i],
                'sog': track['sog'][i],
                'time': track['time'][i],
                'heading': track['heading'],
                'maneuver': track['maneuver'],
                'utc_second': track['utc_second'][i],
            }
            writer.writerow(row)  # Write the row to the CSV file

print(f"All tracks have been combined and written to {csv_filename}")
```
{% endcode %}

We can check the output CSV file now:

{% code lineNumbers="true" %}
```
mmsi 	time 	lon 	lat 	cog 	sog 	utc_second 	heading 	rot 	maneuver
0 	210108000 	1672545711 	-63.645 	44.68833 	173 	0.0 	0 	0.0 	0.0 	False
1 	210108000 	1672545892 	-63.645 	44.68833 	208 	0.0 	0 	0.0 	0.0 	False
2 	210108000 	1672546071 	-63.645 	44.68833 	176 	0.0 	0 	0.0 	0.0 	False
3 	210108000 	1672546250 	-63.645 	44.68833 	50 	0.0 	0 	0.0 	0.0 	False
4 	210108000 	1672546251 	-63.645 	44.68833 	50 	0.0 	0 	0.0 	0.0 	False
```
{% endcode %}

