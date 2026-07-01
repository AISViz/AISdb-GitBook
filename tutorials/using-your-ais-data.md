# 🔐 Using Your AIS Data

In addition to accessing data stored on the AISdb server, you can download open-source AIS data or import your datasets for processing and analysis using AISdb. <mark style="background-color:yellow;">This tutorial guides you through downloading AIS data from popular websites, creating SQLite and PostgreSQL databases compatible with AISdb, and establishing database connections.</mark> We provide two examples: [_Downloading and Processing Individual Files_](using-your-ais-data.md#downloading-and-processing-individual-files)_,_ which demonstrates working with small data samples and creating an SQLite database, and [_Pipeline for Bulk File Downloads and Database Integration_](using-your-ais-data.md#pipeline-for-bulk-file-downloads-and-database-integration)_,_ which outlines our approach to handling multiple data file downloads and creating a PostgreSQL database.

## Data Source

The U.S. vessel traffic data across user-defined geographies and periods are available at [MarineCadastre](https://hub.marinecadastre.gov/pages/vesseltraffic). This resource offers comprehensive AIS data that can be accessed for various maritime analysis purposes. We can tailor the dataset based on research needs by selecting specific regions and timeframes.

## Downloading and Processing Individual Files

In the following example, we will show how to download and process a single data file and import the data to a newly created SQLite database.

First, download the AIS data of the day using the curl command:

{% code lineNumbers="true" %}
```bash
curl -o ./data/AIS_2020_01_01.zip https://coast.noaa.gov/htdata/CMSP/AISDataHandler/2020/AIS_2020_01_01.zip
```
{% endcode %}

Then, extract the downloaded ZIP file to a specific path:

{% code lineNumbers="true" %}
```bash
unzip ./data/AIS_2020_01_01.zip -d ./data/
```
{% endcode %}

We will look into the number of columns in the downloaded CSV file.

{% code lineNumbers="true" %}
```python
import pandas as pd

# Read CSV file in pandas dataframe
df_ = pd.read_csv("./data/AIS_2020_01_01.csv", parse_dates=["BaseDateTime"])

print(df_.columns)
```
{% endcode %}

{% code lineNumbers="true" %}
```python
Index(['MMSI', 'BaseDateTime', 'LAT', 'LON', 'SOG', 'COG', 'Heading',
       'VesselName', 'IMO', 'CallSign', 'VesselType', 'Status',
       'Length', 'Width', 'Draft', 'Cargo', 'TransceiverClass'],
      dtype='object')
```
{% endcode %}

The required columns for AISdb have specific names and may differ from the imported dataset. Therefore, let's define the exact list of columns needed.

{% code lineNumbers="true" %}
```
list_of_headers_ = ["MMSI","Message_ID","Repeat_indicator","Time","Millisecond","Region","Country","Base_station","Online_data","Group_code","Sequence_ID","Channel","Data_length","Vessel_Name","Call_sign","IMO","Ship_Type","Dimension_to_Bow","Dimension_to_stern","Dimension_to_port","Dimension_to_starboard","Draught","Destination","AIS_version","Navigational_status","ROT","SOG","Accuracy","Longitude","Latitude","COG","Heading","Regional","Maneuver","RAIM_flag","Communication_flag","Communication_state","UTC_year","UTC_month","UTC_day","UTC_hour","UTC_minute","UTC_second","Fixing_device","Transmission_control","ETA_month","ETA_day","ETA_hour","ETA_minute","Sequence","Destination_ID","Retransmit_flag","Country_code","Functional_ID","Data","Destination_ID_1","Sequence_1","Destination_ID_2","Sequence_2","Destination_ID_3","Sequence_3","Destination_ID_4","Sequence_4","Altitude","Altitude_sensor","Data_terminal","Mode","Safety_text","Non-standard_bits","Name_extension","Name_extension_padding","Message_ID_1_1","Offset_1_1","Message_ID_1_2","Offset_1_2","Message_ID_2_1","Offset_2_1","Destination_ID_A","Offset_A","Increment_A","Destination_ID_B","offsetB","incrementB","data_msg_type","station_ID","Z_count","num_data_words","health","unit_flag","display","DSC","band","msg22","offset1","num_slots1","timeout1","Increment_1","Offset_2","Number_slots_2","Timeout_2","Increment_2","Offset_3","Number_slots_3","Timeout_3","Increment_3","Offset_4","Number_slots_4","Timeout_4","Increment_4","ATON_type","ATON_name","off_position","ATON_status","Virtual_ATON","Channel_A","Channel_B","Tx_Rx_mode","Power","Message_indicator","Channel_A_bandwidth","Channel_B_bandwidth","Transzone_size","Longitude_1","Latitude_1","Longitude_2","Latitude_2","Station_Type","Report_Interval","Quiet_Time","Part_Number","Vendor_ID","Mother_ship_MMSI","Destination_indicator","Binary_flag","GNSS_status","spare","spare2","spare3","spare4"]
```
{% endcode %}

Next, we update the name of columns in the existing dataframe `df_` and change the time format as required. The timestamp of an AIS message is represented by `BaseDateTime` in the default format `YYYY-MM-DDTHH:MM:SS`. For AISdb, however, the time is represented in UNIX format. We now read the CSV and apply the necessary changes to the date format:

{% code lineNumbers="true" %}
```python
# Take the first 40,000 records from the original dataframe
df = df_.iloc[0:40000]

# Create a new dataframe with the specified headers
df_new = pd.DataFrame(columns=list_of_headers_)

# Populate the new dataframe with formatted data from the original dataframe
df_new['Time'] = pd.to_datetime(df['BaseDateTime']).dt.strftime('%Y%m%d_%H%M%S')
df_new['Latitude'] = df['LAT']
df_new['Longitude'] = df['LON']
df_new['Vessel_Name'] = df['VesselName']
df_new['Call_sign'] = df['CallSign']
df_new['Ship_Type'] = df['VesselType'].fillna(0).astype(int)
df_new['Navigational_status'] = df['Status']
df_new['Draught'] = df['Draft']
df_new['Message_ID'] = 1  # Mark all messages as dynamic by default
df_new['Millisecond'] = 0

# Transfer additional columns from the original dataframe, if they exist
for col_n in df_new:
    if col_n in df.columns:
        df_new[col_n] = df[col_n]

# Extract static messages for each unique vessel
filtered_df = df_new[df_new['Ship_Type'].notnull() & (df_new['Ship_Type'] != 0)]
filtered_df = filtered_df.drop_duplicates(subset='MMSI', keep='first')
filtered_df = filtered_df.reset_index(drop=True)
filtered_df['Message_ID'] = 5  # Mark these as static messages

# Merge dynamic and static messages into a single dataframe
df_new = pd.concat([filtered_df, df_new])

# Save the final dataframe to a CSV file
# The quoting parameter is necessary because the csvreader reads each column value as a string by default
df_new.to_csv("./data/AIS_2020_01_01_aisdb.csv", index=False, quoting=1)
```
{% endcode %}

In the code, we can see that we have mapped the column named accordingly. Additionally, the data type of some columns has also been changed. Additionally, the nm4 file usually contains raw messages, separating static messages from dynamic ones. However, the MarineCadastre Data does not have such a Message\_ID to indicate the type. Thus, adding static messages is necessary for database creation so that a table related to metadata is created.

Let's process the CSV to create an SQLite database using the aisdb package.

{% code lineNumbers="true" %}
```python
import aisdb

# Establish a connection to the SQLite database and decode messages from the CSV file
with aisdb.SQLiteDBConn('./data/test_decode_msgs.db') as dbconn:
        aisdb.decode_msgs(filepaths=["./data/AIS_2020_01_01_aisdb.csv"],
                          dbconn=dbconn, source='Testing', verbose=True)
```
{% endcode %}

{% code lineNumbers="true" %}
```
generating file checksums...
checking file dates...
creating tables and dropping table indexes...
Memory: 20.65GB remaining.  CPUs: 12.  Average file size: 49.12MB  Spawning 4 workers
saving checksums...
processing ./data/AIS_2020_01_01_aisdb.csv
AIS_2020_01_01_aisdb.csv                                         count:   49323    elapsed:    0.27s    rate:   183129 msgs/s
cleaning temporary data...
aggregating static reports into static_202001_aggregate...
```
{% endcode %}

A SQLite database has been created now.&#x20;

{% code lineNumbers="true" %}
```bash
sqlite3 ./data/test_decode_msgs.db

sqlite> .tables
ais_202001_dynamic       coarsetype_ref           static_202001_aggregate
ais_202001_static        hashmap 
```
{% endcode %}

If prefer to progress to PostgreSQL database, defining postgresql string and progress with database connection:

```
// Some code
```









## Pipeline for Bulk File Downloads and Database Integration

This section provides an example of downloading and processing multiple files, creating a PostgreSQL database, and loading data into tables. The steps are outlined in a series of pipeline scripts available in this [GitHub repository](https://github.com/AISViz/NOAA-AIS-Integrator), which should be executed in the order indicated by their numbers.

### AIS Data Download and Extraction

The first script, `0-download-ais.py`, allows you to download AIS data from [MarineCadastre](https://hub.marinecadastre.gov/pages/vesseltraffic) by specifying your needed years. If no years are specified, the script will default to downloading data for 2023. The downloaded ZIP files will be stored in a `/data` folder created in your current working directory. The second script, `1-zip2csv.py`, extracts the CSV files from the downloaded ZIP files in `/data` and saves them in a new directory named `/zip`.&#x20;

To download and extract the data, simply run the two scripts in sequence:

{% code lineNumbers="true" %}
```sh
python 0-download-ais.py
python 1-zip2csv.py
```
{% endcode %}

### Preprocessing - Merge and Deduplication

After downloading and extracting the AIS data, the `2-merge.py` script consolidates the daily CSV files into monthly files while the `3-deduplicate.py` script removes duplicate rows, retaining unique AIS messages. To perform the execution, simply run:

{% code lineNumbers="true" %}
```bash
python 2-merge.py
python 3-deduplicate.py
```
{% endcode %}

The output of these two scripts will be cleaned CSV files, which will be stored in a new folder named `/merged` on your working directory.

### PostgreSQL Database Creation and Data Loading to Tables

The final script, `4-postgresql-database.py`, creates a PostgreSQL database with a specified name. To do this, the script connects to a PostgreSQL server, requiring you to provide your username and password to establish the connection. After creating the database, the script verifies that the number of columns in the CSV files matches the headers. The script creates a corresponding table in the database for each CSV file and loads the data into it. To run this script, you need to provide three command-line arguments: `-dbname` for the new database name, `-user` for your PostgreSQL username, and `-password` for your PostgreSQL password. Additionally, there are two optional arguments: `-host` (default is `localhost`) and `-port` (default is `5432`), you can adjust the `-host` and `-port` values if your PostgreSQL server is running on a different host or port.

{% code lineNumbers="true" %}
```bash
python 4-postgresql-database.py -dbname DBNAME -user USERNAME -password PASSWORD [-host HOST] [-port PORT]
```
{% endcode %}

When the program prompts that the task is finished, you may check the created database and loaded tables by connecting to the PostgreSQL server and using the `psql` command-line interface:

{% code lineNumbers="true" %}
```bash
psql -U USERNAME -d DBNAME -h localhost -p 5432
```
{% endcode %}

Once connected, you can list all tables in the database by running the `\dt` command. In our example using 2023 AIS data (default download), the tables will appear as follows:

{% code lineNumbers="true" %}
```
ais_pgdb=# \dt
           List of relations
 Schema |    Name     | Type  |  Owner  
--------+-------------+-------+----------
 public | ais_2023_01 | table | postgres
 public | ais_2023_02 | table | postgres
 public | ais_2023_03 | table | postgres
 public | ais_2023_04 | table | postgres
 public | ais_2023_05 | table | postgres
 public | ais_2023_06 | table | postgres
 public | ais_2023_07 | table | postgres
 public | ais_2023_08 | table | postgres
 public | ais_2023_09 | table | postgres
 public | ais_2023_10 | table | postgres
 public | ais_2023_11 | table | postgres
 public | ais_2023_12 | table | postgres
(12 rows) 
```
{% endcode %}
