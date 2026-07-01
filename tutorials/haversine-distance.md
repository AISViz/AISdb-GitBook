# 🌎 Haversine Distance

**AISdb** includes a function called `aisdb.gis.delta_meters` that <mark style="background-color:yellow;">calculates the Haversine distance in meters between consecutive positions within a vessel track.</mark> This function is essential for analyzing vessel movement patterns and ensuring accurate distance calculations on the Earth's curved surface. It is also integrated into the [denoising encoder](https://aisdb.meridian.cs.dal.ca/doc/api/aisdb.denoising_encoder.html#aisdb.denoising_encoder.encode_greatcircledistance), which compares distances against a threshold to aid in the data-cleaning process.

Here is an example of calculating the Haversine distance between each pair of consecutive points on a track:

{% code lineNumbers="true" %}
```python
import aisdb
import numpy as np
from aisdb.gis import dt_2_epoch
from datetime import datetime

y1, x1 = 44.57039426840729, -63.52931373766157
y2, x2 = 44.51304767533133, -63.494075674952555
y3, x3 = 44.458038982492134, -63.535634138077945
y4, x4 = 44.393941339104074, -63.53826396955358
y5, x5 = 44.14245580737021, -64.16608964280064

t1 = dt_2_epoch( datetime(2021, 1, 1, 1) )
t2 = dt_2_epoch( datetime(2021, 1, 1, 2) )
t3 = dt_2_epoch( datetime(2021, 1, 1, 3) )
t4 = dt_2_epoch( datetime(2021, 1, 1, 4) )
t5 = dt_2_epoch( datetime(2021, 1, 1, 7) )

# Create a sample track
tracks_short = [
    dict(
        lon=np.array([x1, x2, x3, x4, x5]),
        lat=np.array([y1, y2, y3, y4, y5]),
        time=np.array([t1, t2, t3, t4, t5]),
        mmsi=123456789,
        dynamic=set(['lon', 'lat', 'time']),
        static=set(['mmsi'])
    )
]

# Calculate the Haversine distance
for track in tracks_short:
    print(aisdb.gis.delta_meters(track))
```
{% endcode %}

{% code lineNumbers="true" %}
```
[ 6961.401286 6948.59446128 7130.40147082 57279.94580704]
```
{% endcode %}

If we visualize this track on the map, we can observe:

<figure><img src="../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>
