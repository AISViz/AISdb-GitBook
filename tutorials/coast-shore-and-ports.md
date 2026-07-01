---
description: >-
  Extracting distance features from and to points-of-interest using raster
  files.
---

# 🏝️ Coast, shore, and ports

The distances of a vessel from the nearest shore, coast, and port are essential to perform particular tasks such as vessel behavior analysis, environmental monitoring, and maritime safety assessments. **AISdb** offers functions to acquire these distances for specific vessel positions. In this tutorial, we provide examples of calculating the distance in kilometers from shore and from the nearest port for a given point.

First, we create a sample track:

{% code lineNumbers="true" %}
```python
import aisdb
from aisdb.gis import dt_2_epoch

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

# creating a sample track
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
```
{% endcode %}

Here is what the sample track looks like:

<figure><img src="../.gitbook/assets/image (31).png" alt=""><figcaption><p>Sample track created for distance to shore and port calculation</p></figcaption></figure>

## Distance from shore

The class [`aisdb.webdata.shore_dist.ShoreDist`](https://aisdb.meridian.cs.dal.ca/doc/api/aisdb.webdata.shore_dist.html#aisdb.webdata.shore_dist.ShoreDist) is used to calculate the nearest distance to shore, along with a raster file containing shore distance data. Currently, calling the `get_distance` function in `ShoreDist` will automatically download the shore distance raster file from our server. The function then merges the tracks in the provided track list, creates a new key, "km\_from\_shore", and stores the shore distance as the value for this key.

{% code lineNumbers="true" %}
```python
from aisdb.webdata.shore_dist import ShoreDist

with ShoreDist(data_dir="./testdata/") as sdist:
        # Getting distance from shore for each point in the track
        for track in sdist.get_distance(tracks_short):
            assert 'km_from_shore' in track['dynamic']
            assert 'km_from_shore' in track.keys()
            print(track['km_from_shore'])
```
{% endcode %}

{% code lineNumbers="true" %}
```
[ 1  3  2  9 14]
```
{% endcode %}

## Distance from coast

Similar to acquiring the distance from shore, `CoastDist` is implemented to obtain the distance between the given track positions and the coastline.

{% code lineNumbers="true" %}
```python
from aisdb.webdata.shore_dist import CoastDist

with CoastDist(data_dir="./testdata/") as cdist:
        # Getting distance from the coast for each point in the track
        for track in cdist.get_distance(tracks_short):
            assert 'km_from_coast' in track['dynamic']
            assert 'km_from_coast' in track.keys()
            print(track['km_from_coast'])
```
{% endcode %}

{% code lineNumbers="true" %}
```
[ 1  3  2  8 13]
```
{% endcode %}

## Distance from port

Like the distances from the coast and shore, the [`aisdb.webdata.shore_dist.PortDist`](https://aisdb.meridian.cs.dal.ca/doc/api/aisdb.webdata.shore_dist.html#aisdb.webdata.shore_dist.PortDist) class determines the distance between the track positions and the nearest ports.

{% code lineNumbers="true" %}
```python
from aisdb.webdata.shore_dist import PortDist

with PortDist(data_dir="./testdata/") as pdist:
        # Getting distance from the port for each point in the track
        for track in pdist.get_distance(tracks_short):
            assert 'km_from_port' in track['dynamic']
            assert 'km_from_port' in track.keys()
            print(track['km_from_port'])
```
{% endcode %}

{% code lineNumbers="true" %}
```
[ 4.72144175  7.47747231  4.60478449 11.5642271  28.62511253]
```
{% endcode %}
