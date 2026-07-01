# 🚿 Data Cleaning

A common issue with AIS data is noise, where multiple vessels may broadcast using the same identifier simultaneously. **AISdb** incorporates data cleaning techniques to remove noise from vessel track data. For more details:

**Denoising with Encoder:** The [`aisdb.denoising_encoder.encode_greatcircledistance()`](https://aisdb.meridian.cs.dal.ca/doc/api/aisdb.denoising_encoder.html#aisdb.denoising_encoder.encode_greatcircledistance) function checks the approximate distance between each vessel’s position. It separates vectors where a vessel couldn’t reasonably travel using the most direct path, such as speeds over 50 knots.

**Distance and Speed Thresholds: D**istance and speed thresholds limit the maximum distance or time between messages that can be considered continuous.

**Scoring and Segment Concatenation:** A score is computed for each position delta, with sequential messages nearby at shorter intervals given a higher score. This score is calculated by dividing the Haversine distance by elapsed time. Any deltas with a score not reaching the minimum threshold are considered the start of a new segment. New segments are compared to the end of existing segments with the same vessel identifier; if the score exceeds the minimum, they are concatenated. If multiple segments meet the minimum score, the new segment is concatenated to the existing segment with the highest score.

Processing functions may be executed in sequence as a processing chain or pipeline, so after segmenting the individual voyages, results can be input into the encoder to remove noise and correct for vessels with duplicate identifiers effectively.

{% code lineNumbers="true" %}
```python
import aisdb
from datetime import datetime, timedelta
from aisdb import DBConn, DBQuery, DomainFromPoints

dbpath='YOUR_DATABASE.db' # Define the path to your database

# Set the start and end times for the query
start_time = datetime.strptime("2018-01-01 00:00:00", '%Y-%m-%d %H:%M:%S')
end_time = datetime.strptime("2018-01-02 00:00:00", '%Y-%m-%d %H:%M:%S')

# A circle with a 100km radius around the location point
domain = DomainFromPoints(points=[(-63.6, 44.6)], radial_distances=[50000])

maxdelta = timedelta(hours=24)  # the maximum time interval
distance_threshold = 20000      # the maximum allowed distance (meters) between consecutive AIS messages
speed_threshold = 50            # the maximum allowed vessel speed in consecutive AIS messages
minscore = 1e-6                 # the minimum score threshold for track segment validation

with aisdb.SQLiteDBConn(dbpath=dbpath) as dbconn:
    qry = aisdb.DBQuery(
        dbconn=dbconn, start=start_time, end=end_time,
        callback=aisdb.database.sqlfcn_callbacks.in_timerange_validmmsi,
    )
    rowgen = qry.gen_qry()
    tracks = aisdb.track_gen.TrackGen(rowgen, decimate=False)
    
    # Split the tracks into segments based on the maximum time interval
    track_segments = aisdb.split_timedelta(tracks, maxdelta)
    
    # Encode the track segments to clean and validate the track data
    tracks_encoded = aisdb.encode_greatcircledistance(track_segments, 
                                                      distance_threshold=distance_threshold, 
                                                      speed_threshold=speed_threshold, 
                                                      minscore=minscore)
    tracks_colored = color_tracks(tracks_encoded)
    
    aisdb.web_interface.visualize(
        tracks_colored,
        domain=domain,
        visualearth=True,
        open_browser=True,
    )
```
{% endcode %}

After segmentation and encoding, the tracks are shown as:

<figure><img src="../.gitbook/assets/Screenshot from 2024-08-07 16-37-54.png" alt=""><figcaption><p>Queried vessel tracks after applying track segmentation and encoder (distance threshold=20km, speed threshold=50knots)</p></figcaption></figure>

For comparison, this is a shot of tracks before cleaning:

<figure><img src="../.gitbook/assets/image (28).png" alt=""><figcaption><p>Queried vessel tracks before cleaning</p></figcaption></figure>
