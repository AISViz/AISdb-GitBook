# 🚤 Vessel Speed

In **AISdb**, the speed of a vessel is calculated using the `aisdb.gis.delta_knots` function, which <mark style="background-color:yellow;">computes the</mark> <mark style="background-color:yellow;"></mark>_<mark style="background-color:yellow;">speed over ground (SOG)</mark>_ <mark style="background-color:yellow;"></mark><mark style="background-color:yellow;">in knots</mark> between consecutive positions within a given track. This calculation is important for the [denoising encoder](https://aisdb.meridian.cs.dal.ca/doc/api/aisdb.denoising_encoder.html#aisdb.denoising_encoder.encode_greatcircledistance), as it compares the vessel's speed against a set threshold to aid in the data cleaning process.

Vessel speed calculation requires the **distance** the vessel has traveled between two consecutive positions and the **time interval**. This distance is computed using the [haversine distance](https://aisdb.meridian.cs.dal.ca/doc/api/aisdb.gis.html#aisdb.gis.delta_meters) function, and the time interval is simply the difference in timestamps between the two consecutive AIS position reports. The speed is then computed using the formula:

$$
Speed(knot) = \frac{Haversine Distance}{Time} \times 1.9438445
$$

The factor `1.9438445` converts the speed from meters per second to knots, the standard speed unit used in maritime contexts.

With the example track we created in [_Haversine Distance_](haversine-distance.md), we can calculate the vessel speed between each two consecutive positions:

{% code lineNumbers="true" %}
```python
import aisdb
import numpy as np
from datetime import datetime
from aisdb.gis import dt_2_epoch

# Generate example track
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
        mmsi=123456789,
        lon=np.array([x1, x2, x3, x4, x5]),
        lat=np.array([y1, y2, y3, y4, y5]),
        time=np.array([t1, t2, t3, t4, t5]),
        dynamic=set(['lon', 'lat', 'time']),
        static=set(['mmsi'])
    )
]

# Calculate the vessel speed in knots
for track in tracks_short:
    print(aisdb.gis.delta_knots(track))
```
{% endcode %}

{% code lineNumbers="true" %}
```
[3.7588560005768947 3.7519408684140214 3.8501088005116215 10.309565520121597]
```
{% endcode %}
