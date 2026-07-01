---
cover: >-
  https://images.unsplash.com/photo-1544383835-bda2bc66a55d?crop=entropy&cs=srgb&fm=jpg&ixid=M3wxOTcwMjR8MHwxfHNlYXJjaHwzfHxkYXRhYmFzZXxlbnwwfHx8fDE3MjMyNjIyOTJ8MA&ixlib=rb-4.0.3&q=85
coverY: 47.15
---

# 🗄️ SQL Database

### Table Naming

When loading data into the database, messages will be sorted into SQL tables determined by the message type and month. The names of these tables follow the following format, which `{YYYYMM}` indicates the table year and month in the format YYYYMM.

{% code lineNumbers="true" %}
```
ais_{YYYYMM}_static  # table with static AIS messages
ais_{YYYYMM}_dynamic # table with dynamic AIS message
```
{% endcode %}

Some additional tables containing computed data may be created depending on the indexes used. For example, an aggregate of vessel static data by month or a virtual table is used as a covering index.&#x20;

{% code lineNumbers="true" %}
```
static_{YYYYMM}_aggregate # table of aggregated static vessel data
```
{% endcode %}

Additional tables are also included for storing data not directly derived from AIS message reports.

{% code lineNumbers="true" %}
```
coarsetype_ref # a reference table that maps numeric ship type codes to their descriptions hashmap
```
{% endcode %}

For quick reference to data types and detailed explanations of these table entries, please see the [Detailed Table Description](sql-database.md#detailed-table-description).

### Custom SQL Queries

In addition to querying the database using [`DBQuery`](https://aisdb.meridian.cs.dal.ca/doc/api/aisdb.database.dbqry.html) module, there is an option to customize the query with your own SQL code.&#x20;

Example of listing all the tables in your database:

{% code lineNumbers="true" %}
```python
import sqlite3

dbpath='YOUR_DATABASE.db' # Define the path to your database

# Connect to the database
connection = sqlite3.connect(dbpath)

# Create a cursor object
cursor = connection.cursor()

# Query to list all tables
query = "SELECT name FROM sqlite_master WHERE type='table';"
cursor.execute(query)

# Fetch the results
tables = cursor.fetchall()

# Print the names of the tables
print("Tables in the database:")
for table in tables:
    print(table[0])

# Close the connection
connection.close()
```
{% endcode %}

As messages are separated into tables by message type and month, queries spanning multiple message types or months should use UNIONs and JOINs to combine results as appropriate.&#x20;

Example of querying tables with \`JOIN\`:

{% code lineNumbers="true" %}
```sql
import sqlite3

# Connect to the database
connection = sqlite3.connect('YOUR_DATABASE.db')

# Create a cursor object
cursor = connection.cursor()

# Define the JOIN SQL query
query = f"""
SELECT
    d.mmsi, 
    d.time,
    d.longitude,
    d.latitude,
    d.sog,
    d.cog,
    s.vessel_name,
    s.ship_type
FROM ais_{YYYYMM}_dynamic d
LEFT JOIN ais_{YYYYMM}_static s ON d.mmsi = s.mmsi
WHERE d.time BETWEEN 1707033659 AND 1708176856  -- Filter by time range
  AND d.longitude BETWEEN -68 AND -56           -- Filter by geographical area
  AND d.latitude BETWEEN 45 AND 51.5;
"""

# Execute the query
cursor.execute(query)

# Fetch the results
results = cursor.fetchall()

# Print the results
for row in results:
    print(row)

# Close the connection
connection.close()
```
{% endcode %}

More information about SQL queries can be looked up from [online tutorials](https://sqlbolt.com/).

The R\* tree virtual tables should be queried for AIS position reports instead of the default tables. Query performance can be significantly improved using the R\* tree index when restricting output to a narrow range of MMSIs, timestamps, longitudes, and latitudes. However, querying a wide range will not yield much benefit. If custom indexes are required for specific manual queries, these should be defined on message tables 1\_2\_3, 5, 18, and 24 directly instead of upon the virtual tables.

Timestamps are stored as epoch minutes in the database. To facilitate querying the database manually, use the `dt_2_epoch()` function to convert datetime values to epoch minutes and the `epoch_2_dt()` function to convert epoch minutes back to datetime values. Here is how you can use `dt_2_epoch()` with the example above:

{% code lineNumbers="true" %}
```python
from aisdb.gis import dt_2_epoch

# Define the datetime range
start_datetime = datetime(2018, 1, 1, 0, 0, 0)
end_datetime = datetime(2018, 1, 1, 1, 59, 59)

# Convert datetime to epoch time
start_epoch = dt_2_epoch(start_datetime)
end_epoch = dt_2_epoch(end_datetime)

# Connect to the database
connection = sqlite3.connect('YOUR_DATABASE.db')

# Create a cursor object
cursor = connection.cursor()

# Define the JOIN SQL query using an epoch time range
query = f"""
SELECT
    d.mmsi, 
    d.time,
    d.longitude,
    d.latitude,
    d.sog,
    d.cog,
    s.vessel_name,
    s.ship_type
FROM ais_201801_dynamic d
LEFT JOIN ais_201801_static s ON d.mmsi = s.mmsi
WHERE d.time BETWEEN {start_epoch} AND {end_epoch}  -- Filter by time range
  AND d.longitude BETWEEN -68 AND -56           -- Filter by geographical area
  AND d.latitude BETWEEN 45 AND 51.5;
"""

# Execute the query
cursor.execute(query)

# Fetch the results
results = cursor.fetchall()

# Print the results
for row in results:
    print(row)

# Close the connection
connection.close()
```
{% endcode %}

For more examples, please see the SQL code in [`aisdb_sql/`](https://github.com/AISViz/AISdb/tree/master/aisdb/aisdb_sql) that is used to create database tables and associated queries.

### Detailed Table Description

#### `ais_{YYYYMM}_dynamic` tables

<table><thead><tr><th width="202">Column</th><th width="157">Data Type</th><th>Description</th></tr></thead><tbody><tr><td><code>mmsi</code></td><td><code>INTEGER</code></td><td>Maritime Mobile Service Identity, a unique identifier for vessels.</td></tr><tr><td><code>time</code></td><td><code>INTEGER</code></td><td>Timestamp of the AIS message, in epoch seconds.</td></tr><tr><td><code>longitude</code></td><td><code>REAL</code></td><td>Longitude of the vessel in decimal degrees.</td></tr><tr><td><code>latitude</code></td><td><code>REAL</code></td><td>Latitude of the vessel in decimal degrees.</td></tr><tr><td><code>rot</code></td><td><code>REAL</code></td><td>Rate of turn, indicating how fast the vessel is turning.</td></tr><tr><td><code>sog</code></td><td><code>REAL</code></td><td>Speed over ground, in knots.</td></tr><tr><td><code>cog</code></td><td><code>REAL</code></td><td>Course over ground, in degrees.</td></tr><tr><td><code>heading</code></td><td><code>REAL</code></td><td>Heading of the vessel, in degrees.</td></tr><tr><td><code>maneuver</code></td><td><code>BOOLEAN</code></td><td>Indicator for whether the vessel is performing a special maneuver.</td></tr><tr><td><code>utc_second</code></td><td><code>INTEGER</code></td><td>Second of the UTC timestamp when the message was generated.</td></tr><tr><td><code>source</code></td><td><code>TEXT</code></td><td>Source of the AIS data.</td></tr></tbody></table>

#### `ais_{YYYYMM}_static` tables

<table><thead><tr><th width="205">Column</th><th width="157">Data Type</th><th>Description</th></tr></thead><tbody><tr><td><code>mmsi</code></td><td><code>INTEGER</code></td><td>Maritime Mobile Service Identity, a unique identifier for vessels.</td></tr><tr><td><code>time</code></td><td><code>INTEGER</code></td><td>Timestamp of the AIS message, in epoch seconds.</td></tr><tr><td><code>vessel_name</code></td><td><code>TEXT</code></td><td>Name of the vessel.</td></tr><tr><td><code>ship_type</code></td><td><code>INTEGER</code></td><td>Numeric code representing the type of ship.</td></tr><tr><td><code>call_sign</code></td><td><code>TEXT</code></td><td>International radio call sign of the vessel.</td></tr><tr><td><code>imo</code></td><td><code>INTEGER</code></td><td>International Maritime Organization number, another unique vessel identifier.</td></tr><tr><td><code>dim_bow</code></td><td><code>INTEGER</code></td><td>Distance from the AIS transmitter to the bow (front) of the vessel.</td></tr><tr><td><code>dim_stern</code></td><td><code>INTEGER</code></td><td>Distance from the AIS transmitter to the stern (back) of the vessel.</td></tr><tr><td><code>dim_port</code></td><td><code>INTEGER</code></td><td>Distance from the AIS transmitter to the port (left) side of the vessel.</td></tr><tr><td><code>dim_star</code></td><td><code>INTEGER</code></td><td>Distance from the AIS transmitter to the starboard (right) side of the vessel.</td></tr><tr><td><code>draught</code></td><td><code>REAL</code></td><td>Maximum depth of the vessel's hull below the waterline, in meters.</td></tr><tr><td><code>destination</code></td><td><code>TEXT</code></td><td>Destination port or location where the vessel is heading.</td></tr><tr><td><code>ais_version</code></td><td><code>INTEGER</code></td><td>AIS protocol version used by the vessel.</td></tr><tr><td><code>fixing_device</code></td><td><code>TEXT</code></td><td>Type of device used for fixing the vessel's position (e.g., GPS).</td></tr><tr><td><code>eta_month</code></td><td><code>INTEGER</code></td><td>Estimated time of arrival month.</td></tr><tr><td><code>eta_day</code></td><td><code>INTEGER</code></td><td>Estimated time of arrival day.</td></tr><tr><td><code>eta_hour</code></td><td><code>INTEGER</code></td><td>Estimated time of arrival hour.</td></tr><tr><td><code>eta_minute</code></td><td><code>INTEGER</code></td><td>Estimated time of arrival minute.</td></tr><tr><td><code>source</code></td><td><code>TEXT</code></td><td>Source of the AIS data (e.g., specific AIS receiver or data provider).</td></tr></tbody></table>

#### &#x20;`static_{YYYYMM}_aggregate` tables

<table><thead><tr><th width="206">Column</th><th width="157">Data Type</th><th>Description</th></tr></thead><tbody><tr><td><code>mmsi</code></td><td><code>INTEGER</code></td><td>Maritime Mobile Service Identity, a unique identifier for vessels.</td></tr><tr><td><code>imo</code></td><td><code>INTEGER</code></td><td>International Maritime Organization number, another unique vessel identifier.</td></tr><tr><td><code>vessel_name</code></td><td><code>TEXT</code></td><td>Name of the vessel.</td></tr><tr><td><code>ship_type</code></td><td><code>INTEGER</code></td><td>Numeric code representing the type of ship.</td></tr><tr><td><code>call_sign</code></td><td><code>TEXT</code></td><td>International radio call sign of the vessel.</td></tr><tr><td><code>dim_bow</code></td><td><code>INTEGER</code></td><td>Distance from the AIS transmitter to the bow (front) of the vessel.</td></tr><tr><td><code>dim_stern</code></td><td><code>INTEGER</code></td><td>Distance from the AIS transmitter to the stern (back) of the vessel.</td></tr><tr><td><code>dim_port</code></td><td><code>INTEGER</code></td><td>Distance from the AIS transmitter to the port (left) side of the vessel.</td></tr><tr><td><code>dim_star</code></td><td><code>INTEGER</code></td><td>Distance from the AIS transmitter to the starboard (right) side of the vessel.</td></tr><tr><td><code>draught</code></td><td><code>REAL</code></td><td>Maximum depth of the vessel's hull below the waterline, in meters.</td></tr><tr><td><code>destination</code></td><td><code>TEXT</code></td><td>Destination port or location where the vessel is heading.</td></tr><tr><td><code>eta_month</code></td><td><code>INTEGER</code></td><td>Estimated time of arrival month.</td></tr><tr><td><code>eta_day</code></td><td><code>INTEGER</code></td><td>Estimated time of arrival day.</td></tr><tr><td><code>eta_hour</code></td><td><code>INTEGER</code></td><td>Estimated time of arrival hour.</td></tr><tr><td><code>eta_minute</code></td><td><code>INTEGER</code></td><td>Estimated time of arrival minute.</td></tr></tbody></table>
