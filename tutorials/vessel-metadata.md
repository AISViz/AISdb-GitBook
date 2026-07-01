# ⬇️ Vessel Metadata

This tutorial demonstrates how to access vessel metadata using MMSI and SQLite databases. In many cases, AIS messages do not contain metadata. Therefore, <mark style="background-color:yellow;">this tutorial introduces the built-in functions in AISdb and external APIs to extract detailed vessel information</mark> associated with a specific MMSI from web sources.

## Metadata Download

We introduced two methods implemented in **AISdb** for scraping metadata: using <mark style="background-color:yellow;">session requests</mark> for direct access and employing <mark style="background-color:yellow;">web drivers with browsers</mark> to handle modern websites with dynamic content. Additionally, we provided an example of utilizing a <mark style="background-color:yellow;">third-party API</mark> to access vessel information.

### Session Request

The session request method in Python is a straightforward and efficient approach for retrieving metadata from websites. In AISdb, the `aisdb.webdata._scraper.search_metadata_vesselfinder` function leverages this method to scrape detailed information about vessels based on their MMSI numbers. This function efficiently gathers a range of data, including vessel name, type, flag, tonnage, and navigation status.&#x20;

This is an example of how to use the `search_metadata_vesselfinder` feature in AISdb to scrape data from [VesselFinder](https://www.vesselfinder.com/) website:

{% code lineNumbers="true" %}
```python
from aisdb.webdata._scraper import search_metadata_vesselfinder

MMSI = 228386800
dict_ = search_metadata_vesselfinder(MMSI)

print(dict_)
```
{% endcode %}

{% code lineNumbers="true" %}
```
{'IMO number': '9839131',
 'Vessel Name': 'CMA CGM CHAMPS ELYSEES',
 'Ship type': 'Container Ship',
 'Flag': 'France',
 'Homeport': '-',
 'Gross Tonnage': '236583',
 'Summer Deadweight (t)': '220766',
 'Length Overall (m)': '400',
 'Beam (m)': '61',
 'Draught (m)': '',
 'Year of Build': '2020',
 'Builder': '',
 'Place of Build': '',
 'Yard': '',
 'TEU': '',
 'Crude Oil (bbl)': '-',
 'Gas (m3)': '-',
 'Grain': '-',
 'Bale': '-',
 'Classification Society': '',
 'Registered Owner': '',
 'Owner Address': '',
 'Owner Website': '-',
 'Owner Email': '-',
 'Manager': '',
 'Manager Address': '',
 'Manager Website': '',
 'Manager Email': '',
 'Predicted ETA': '',
 'Distance / Time': '',
 'Course / Speed': '\xa0',
 'Current draught': '16.0 m',
 'Navigation Status': '\nUnder way\n',
 'Position received': '\n22 mins ago \n\n\n',
 'IMO / MMSI': '9839131 / 228386800',
 'Callsign': 'FLZF',
 'Length / Beam': '399 / 61 m'}
```
{% endcode %}

### MarineTraffic API

In addition to metadata scraping, we may also use the available API the data provides. MarineTraffic offers an option to subscribe to its API to access vessel data, forecast voyages, position the vessels, etc. Here is an example of retrieving :

{% code lineNumbers="true" %}
```python
import requests

# Your MarineTraffic API key
api_key = 'your_marine_traffic_api_key'

# List of MMSI numbers you want to query
mmsi = [228386800,
        372351000,
        373416000,
        477003800,
        477282400
]

# Base URL for the MarineTraffic API endpoint
url = f'https://services.marinetraffic.com/api/exportvessels/{api_key}'

# Prepare the API request
params = {
    'shipid': ','.join(mmsi_list),  # Join MMSI list with commas
    'protocol': 'jsono',            # Specify the response format
    'msgtype': 'extended'           # Specify the level of details
}

# Make the API request
response = requests.get(url, params=params)

# Check if the request was successful
if response.status_code == 200:
    vessel_data = response.json()
    
    for vessel in vessel_data:
        print(f"Vessel Name: {vessel.get('NAME')}")
        print(f"MMSI: {vessel.get('MMSI')}")
        print(f"IMO: {vessel.get('IMO')}")
        print(f"Call Sign: {vessel.get('CALLSIGN')}")
        print(f"Type: {vessel.get('TYPE_NAME')}")
        print(f"Flag: {vessel.get('COUNTRY')}")
        print(f"Length: {vessel.get('LENGTH')}")
        print(f"Breadth: {vessel.get('BREADTH')}")
        print(f"Year Built: {vessel.get('YEAR_BUILT')}")
        print(f"Status: {vessel.get('STATUS_NAME')}")
        print('-' * 40)
else:
    print(f"Failed to retrieve data: {response.status_code}")
```
{% endcode %}

## Metadata Storage

If you already have a database containing AIS track data, then vessel metadata can be downloaded and stored in a separate database.

{% code lineNumbers="true" %}
```python
from aisdb import track_gen, decode_msgs, DBQuery, sqlfcn_callbacks, Domain
from datetime import datetime 

dbpath = "/home/database.db"
start = datetime(2021, 11, 1)
end = datetime(2021, 11, 2)

with DBConn(dbpath="/home/data_sample_dynamic.csv.db") as dbconn:
        qry = aisdb.DBQuery(
             dbconn=dbconn, callback=in_timerange,
             start=datetime(2020, 1, 1, hour=0),
             end=datetime(2020, 12, 3, hour=2)
        )
        # A new database will be created if it does not exist to save the downloaded info from MarineTraffic
        traffic_database_path = "/home/traffic_info.db"
       
        # User can select a custom boundary for a query using aisdb.Domain
        qry.check_marinetraffic(trafficDBpath=, boundary={"xmin":-180, "xmax":180, "ymin":-180,  "ymax":180})
        
        rowgen = qry.gen_qry(verbose=True)
        trackgen = track_gen.TrackGen(rowgen, decimate=True)
```
{% endcode %}

