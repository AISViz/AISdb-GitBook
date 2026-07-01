---
description: Trajectory Forecasting with Gate Recurrent Units AutoEncoders
icon: k
---

# AutoEncoders in Keras

## **Introduction** <a href="#introduction" id="introduction"></a>

By the end of this tutorial, you will understand the benefits of using teacher forcing to improve model accuracy, as well as other tweaks to enhance forecasting capabilities. We'll use AutoEncoders, neural networks that learn compressed data representations, to achieve this.

We will guide you through preparing AIS data for training an AutoEncoder, setting up layers, compiling the model, and defining the training process with teacher forcing.

Given the complexity of this task, we will revisit it to explore the benefits of teacher forcing, a technique that can improve sequence-to-sequence learning in neural networks.

This tutorial focuses on Trajectory Forecasting, which predicts an object's future path based on past positions. We will work with AIS messages, a type of temporal data that provides information about vessels' location, speed, and heading over time.

## AISdb Querying <a href="#aisdb-querying" id="aisdb-querying"></a>

**Automatic Identification System (AIS)** messages broadcast essential ship information such as position, speed, and course. The temporal nature of these messages is pivotal for our tutorial, where we'll train an **auto-encoder** neural network for **trajectory forecasting**. This task involves predicting a ship's future path based on its past AIS messages, making it ideal for auto-encoders, which are optimized for learning patterns in sequential data.

```python
def qry_database(dbname, start_time, stop_time):
    d_threshold = 200000  # max distance (in meters) between two messages before assuming separate tracks
    s_threshold = 50  # max speed (in knots) between two AIS messages before splitting tracks
    t_threshold = timedelta(hours=24)  # max time (in hours) between messages of a track
    try:
        with aisdb.DBConn() as dbconn:
            tracks = aisdb.TrackGen(
                aisdb.DBQuery(
                    dbconn=dbconn, dbpath=os.path.join(ROOT, PATH, dbname),
                    callback=aisdb.database.sql_query_strings.in_timerange,
                    start=start_time, end=stop_time).gen_qry(),
                    decimate=False)  # trajectory compression
            tracks = aisdb.split_timedelta(tracks, t_threshold)  # split trajectories by time without AIS message transmission
            tracks = aisdb.encode_greatcircledistance(tracks, distance_threshold=d_threshold, speed_threshold=s_threshold)
            tracks = aisdb.interp_time(tracks, step=timedelta(minutes=5))  # interpolate every n-minutes
            # tracks = vessel_info(tracks, dbconn=dbconn)  # scrapes vessel metadata
            return list(tracks)  # list of segmented pre-processed tracks
    except SyntaxError as e: return []  # no results for query
```

_For querying the entire database at once, use the following code:_

```python
def get_tracks(dbname, start_ddmmyyyy, stop_ddmmyyyy):
    stop_time = datetime.strptime(stop_ddmmyyyy, "%d%m%Y")
    start_time = datetime.strptime(start_ddmmyyyy, "%d%m%Y")
    # returns a list with all tracks from AISdb
    return qry_database(dbname, start_time, stop_time)
```

_For querying the database in batches of hours, use the following code:_

```python
def batch_tracks(dbname, start_ddmmyyyy, stop_ddmmyyyy, hours2batch):
    stop_time = datetime.strptime(stop_ddmmyyyy, "%d%m%Y")
    start_time = datetime.strptime(start_ddmmyyyy, "%d%m%Y")
    # yields a list of results every delta_time iterations
    delta_time = timedelta(hours=hours2batch)
    anchor_time, next_time = start_time, start_time + delta_time
    while next_time < stop_time:
        yield qry_database(dbname, anchor_time, next_time)
        anchor_time = next_time
        next_time += delta_time
    # yields a list of final results (if any)
    yield qry_database(dbname, anchor_time, stop_time)
```

Several functions were defined using AISdb, an AIS framework developed by MERIDIAN at Dalhousie University, to efficiently extract AIS messages from SQLite databases. AISdb is designed for effective data storage, retrieval, and preparation for AIS-related tasks. It provides comprehensive tools for interacting with AIS data, including APIs for data reading and writing, parsing AIS messages, and performing various data transformations.

### **Data Visualization**

Our next step is to create a coverage map of _Atlantic Canada_ to visualize our dataset. We will include a 100km radius circle on the map to show the areas of the ocean where vessels can send AIS messages. Although overlapping circles may contain duplicate data from the same MMSI, we have already eliminated those from our dataset. However, messages might still appear incorrectly in inland areas.

```python
# Create the map with specific latitude and longitude limits and a Mercator projection
m = Basemap(llcrnrlat=42, urcrnrlat=52, llcrnrlon=-70, urcrnrlon=-50, projection="merc", resolution="h")

# Draw state, country, coastline borders, and counties
m.drawstates(0.5)
m.drawcountries(0.5)
m.drawcoastlines(0.5)
m.drawcounties(color="gray", linewidth=0.5)

# Fill continents and oceans
m.fillcontinents(color="tan", lake_color="#91A3B0")
m.drawmapboundary(fill_color="#91A3B0")

coordinates = [
    (51.26912, -57.53759), (48.92733, -58.87786),
    (47.49307, -59.41325), (42.54760, -62.17624),
    (43.21702, -60.49943), (44.14955, -60.59600),
    (45.42599, -59.76398), (46.99134, -60.02403)]

# Draw 100km-radius circles
for lat, lon in coordinates:
    radius_in_degrees = 100 / (111.32 * np.cos(np.deg2rad(lat)))
    m.tissot(lon, lat, radius_in_degrees, 100, facecolor="r", edgecolor="k", alpha=0.5)

# Add text annotation with an arrow pointing to the circle
plt.annotate("AIS Coverage", xy=m(lon, lat), xytext=(40, -40),
             textcoords="offset points", ha="left", va="bottom", fontweight="bold",
             arrowprops=dict(arrowstyle="->", color="k", alpha=0.7, lw=2.5))

# Add labels
ocean_labels = {
    "Atlantic Ocean": [(-59, 44), 16],
    "Gulf of\nMaine": [(-67, 44.5), 12],
    "Gulf of St. Lawrence": [(-64.5, 48.5), 11],
}

for label, (coords, fontsize) in ocean_labels.items():
    plt.annotate(label, xy=m(*coords), xytext=(6, 6), textcoords="offset points",
                 fontsize=fontsize, color="#DBE2E9", fontweight="bold")

# Add a scale in kilometers
m.drawmapscale(-67.5, 42.7, -67.5, 42.7, 500, barstyle="fancy", fontsize=8, units="km", labelstyle="simple")

# Set the map title
_ = plt.title("100km-AIS radius-coverage on Atlantic Canada", fontweight="bold")
# The circle diameter is 200km, and it does not match the km scale (approximation)
```

<figure><img src="../.gitbook/assets/image (14).png" alt=""><figcaption><p>AIS Data Cover in the Gulf of St. Lawrence</p></figcaption></figure>

_Loading a shapefile to help us define whether a vessel is on land or in water during the trajectory:_

```python
land_polygons = gpd.read_file(os.path.join(ROOT, SHAPES, "ne_50m_land.shp"))
```

_Check if a given coordinate (latitude, longitude) is on land:_

```python
def is_on_land(lat, lon, land_polygons):
    return land_polygons.contains(Point(lon, lat)).any()
```

_Check if any coordinate of a track is on land:_

```python
def is_track_on_land(track, land_polygons):
    for lat, lon in zip(track["lat"], track["lon"]):
        if is_on_land(lat, lon, land_polygons):
            return True
    return False
```

_Filter out tracks with any point on land for a given MMSI:_

```python
def process_mmsi(item, polygons):
    mmsi, tracks = item
    filtered_tracks = [t for t in tracks if not is_track_on_land(t, polygons)]
    return mmsi, filtered_tracks, len(tracks)
```

_Use a ThreadPoolExecutor to parallelize the processing of MMSIs:_

```python
def process_voyages(voyages, land_polygons):

    # Tracking progress with TQDM
    def process_mmsi_callback(result, progress_bar):
        mmsi, filtered_tracks, _ = result
        voyages[mmsi] = filtered_tracks
        progress_bar.update(1)

    # Initialize the progress bar with the total number of MMSIs
    progress_bar = tqdm(total=len(voyages), desc="MMSIs processed")

    with ThreadPoolExecutor(max_workers=multiprocessing.cpu_count()) as executor:
        # Submit all MMSIs for processing
        futures = {executor.submit(process_mmsi, item, land_polygons): item for item in voyages.items()}

        # Retrieve the results as they become available and update the Voyages dictionary
        for future in as_completed(futures):
            result = future.result()
            process_mmsi_callback(result, progress_bar)

    # Close the progress bar after processing the complete
    progress_bar.close()
    return voyages
file_name = "curated-ais.pkl"
full_path = os.path.join(ROOT, ESRF, file_name)
if not os.path.exists(full_path):
    voyages = process_voyages(voyages, land_polygons)
    pkl.dump(voyages, open(full_path, "wb"))
else: voyages = pkl.load(open(full_path, "rb"))
```

_Count the number of segments per MMSI after removing duplicates and inaccurate track segments:_

```python
voyages_counts = {k: len(voyages[k]) for k in voyages.keys()}
```

### Dataset Preparation

In this analysis, we observe that most MMSIs in the dataset exhibit between 1 and 49 segments during the search period within AISdb. However, a minor fraction of vessels have significantly more segments, with some reaching up to 176. Efficient processing involves categorizing the data by MMSI instead of merely considering its volume. This method allows us to better evaluate the model's ability to discern various movement behaviors from both the same vessel and different ones.

```python
def plot_voyage_segments_distribution(voyages_counts, bar_color="#ba1644"):
    data = pd.DataFrame({"Segments": list(voyages_counts.values())})
    return  alt.Chart(data).mark_bar(color=bar_color).encode(
                alt.X("Segments:Q", bin=alt.Bin(maxbins=90), title="Segments"),
                alt.Y("count(Segments):Q", title="Count", scale=alt.Scale(type="log")))\
            .properties(title="Distribution of Voyage Segments", width=600, height=400)\
            .configure_axisX(titleFontSize=16).configure_axisY(titleFontSize=16)\
            .configure_title(fontSize=18).configure_view(strokeOpacity=0)
alt.data_transformers.enable("default", max_rows=None)
plot_voyage_segments_distribution(voyages_counts).display()
```

To prevent our model from favoring shorter trajectories, we need a balanced mix of short-term and long-term voyages in the training and test sets. We'll categorize trajectories with 30 or more segments as long-term and those with fewer segments as short-term. Implement an 80-20 split strategy to ensure an equitable distribution of both types in the datasets.

```python
long_term_voyages, short_term_voyages = [], []
# Separating voyages
for k in voyages_counts:
    if voyages_counts[k] < 30:
        short_term_voyages.append(k)
    else: long_term_voyages.append(k)
# Shuffling for random distribution
random.shuffle(short_term_voyages)
random.shuffle(long_term_voyages)
```

_Splitting the data respecting the voyage length distribution:_

```python
train_voyage, test_voyage = {}, {}
# Iterate over short-term voyages:
for i, k in enumerate(short_term_voyages):
    if i < int(0.8 * len(short_term_voyages)):
        train_voyage[k] = voyages[k]
    else: test_voyage[k] = voyages[k]
# Iterate over long-term voyages:
for i, k in enumerate(long_term_voyages):
    if i < int(0.8 * len(long_term_voyages)):
        train_voyage[k] = voyages[k]
    else: test_voyage[k] = voyages[k]
```

_Visualizing the distribution of the dataset:_

```python
def plot_voyage_length_distribution(data, title, bar_color, min_time=144, force_print=True):
    total_time = []
    for key in data.keys():
        for track in data[key]:
            if len(track["time"]) > min_time or force_print:
                total_time.append(len(track["time"]))
    plot_data = pd.DataFrame({'Length': total_time})
    chart = alt.Chart(plot_data).mark_bar(color=bar_color).encode(
        alt.Y("count(Length):Q", title="Count", scale=alt.Scale(type="symlog")),
        alt.X("Length:Q", bin=alt.Bin(maxbins=90), title="Length")
    ).properties(title=title, width=600, height=400)\
    .configure_axisX(titleFontSize=16).configure_axisY(titleFontSize=16)\
    .configure_title(fontSize=18).configure_view(strokeOpacity=0)
    print("\n\n")
    return chart
display(plot_voyage_length_distribution(train_voyage, "TRAINING: Distribution of Voyage Length", "#287561"))
display(plot_voyage_length_distribution(test_voyage, "TEST: Distribution of Voyage Length", "#3e57ab"))
```

## Inputs & Outputs

Understanding input and output timesteps and variables is crucial in trajectory forecasting tasks. Trajectory data comprises spatial coordinates and related features that depict an object's movement over time. The aim is to predict future positions of the object based on its historical data and associated features.

* **INPUT\_TIMESTEPS:** This parameter determines the consecutive observations used to predict future trajectories. Its selection impacts the model's ability to capture temporal dependencies and patterns. Too few time steps may prevent the model from capturing all movement dynamics, resulting in inaccurate predictions. Conversely, too many time steps can add noise and complexity, increasing the risk of overfitting.
* **INPUT\_VARIABLES:** Features describe each timestep in the input sequence for trajectory forecasting. These variables can include spatial coordinates, velocities, accelerations, object types, and relevant features that aid in predicting system dynamics. Choosing the right input variables is crucial; irrelevant or redundant ones may confuse the model while missing important variables can result in poor predictions.
* **OUTPUT\_TIMESTEPS:** This parameter sets the number of future time steps the model should predict, known as the prediction horizon. Choosing the right horizon size is critical. Predicting too few timesteps may not serve the application's needs while predicting too many can increase uncertainty and degrade performance. Select a value based on your application's specific requirements and data quality.
* **OUTPUT\_VARIABLES:** In trajectory forecasting, output variables include predicted spatial coordinates and sometimes other relevant features. Reducing the number of output variables can simplify prediction tasks and decrease model complexity. However, this approach might also lead to a less effective model.

Understanding the roles of input and output timesteps and variables is key to developing accurate trajectory forecasting models. By carefully selecting these elements, we can create models that effectively capture object movement dynamics, resulting in more accurate and meaningful predictions across various applications.

For this tutorial, we'll input **4 hours** of data into the model to forecast the next **8 hours** of vessel movement. Consequently, we'll filter out all voyages with less than **12 hours** of AIS messages. By interpolating the messages every `5 minutes`, we require a **minimum of 144 sequential messages** (12 hours at 12 messages/hour).

With data provided by AISdb, we have AIS information, including Longitude, Latitude, Course Over Ground (COG), and Speed Over Ground (SOG), representing a ship's position and movement. _Longitude_ and _Latitude_ specify the ship's location, while **COG** and **SOG** indicate its heading and speed. By using `all features for training` the neural network, our _output_ will be the _Longitude_ and _Latitude_ pair. This methodology allows the model to predict the ship's future positions based on historical data.

```python
INPUT_TIMESTEPS = 48  # 4 hours * 12 AIS messages/h
INPUT_VARIABLES = 4  # Longitude, Latitude, COG, and SOG
OUTPUT_TIMESTEPS = 96  # 8 hours * 12 AIS messages/h
OUTPUT_VARIABLES = 2  # Longitude and Latitude
NUM_WORKERS = multiprocessing.cpu_count()
```

In this tutorial, we'll include AIS data deltas as features, which were excluded in the previous tutorial. Incorporating deltas can help the model capture temporal changes and patterns, enhancing its effectiveness in sequence-to-sequence modeling. Deltas provides information on the rate of change in features, improving the model's accuracy, especially in predicting outcomes that depend on temporal dynamics.

```python
INPUT_VARIABLES *= 2  # Double the features with deltas
```

### **Data Filtering**

```python
def filter_and_transform_voyages(voyages):
    filtered_voyages = {}
    for k, v in voyages.items():
        voyages_track = []
        for voyage in v:
            if len(voyage["time"]) > (INPUT_TIMESTEPS + OUTPUT_TIMESTEPS):
                mtx = np.vstack([voyage["lon"], voyage["lat"],
                                 voyage["cog"], voyage["sog"]]).T
                # Compute deltas
                deltas = np.diff(mtx, axis=0)
                # Add zeros at the first row for deltas
                deltas = np.vstack([np.zeros(deltas.shape[1]), deltas])
                # Concatenate the original matrix with the deltas matrix
                mtx = np.hstack([mtx, deltas])
                voyages_track.append(mtx)
        if len(voyages_track) > 0:
            filtered_voyages[k] = voyages_track
    return filtered_voyages
# Checking how the data behaves for the previously set hyperparameters
display(plot_voyage_length_distribution(train_voyage, "TRAINING: Distribution of Voyage Length", "#287561",
                                        min_time=INPUT_TIMESTEPS + OUTPUT_TIMESTEPS, force_print=False))
display(plot_voyage_length_distribution(test_voyage, "TEST: Distribution of Voyage Length", "#3e57ab",
                                        min_time=INPUT_TIMESTEPS + OUTPUT_TIMESTEPS, force_print=False))
# Filter and transform train and test voyages and prepare for training
train_voyage = filter_and_transform_voyages(train_voyage)
test_voyage = filter_and_transform_voyages(test_voyage)
```

### **Data Statistics**

```python
def print_voyage_statistics(header, voyage_dict):
    total_time = 0
    for mmsi, trajectories in voyage_dict.items():
        for trajectory in trajectories:
            total_time += trajectory.shape[0]
    print(f"{header}")
    print(f"Hours of sequential data: {total_time // 12}.")
    print(f"Number of unique MMSIs: {len(voyage_dict)}.", end=" \n\n")
    return total_time
time_test = print_voyage_statistics("[TEST DATA]", test_voyage)
time_train = print_voyage_statistics("[TRAINING DATA]", train_voyage)
# We remained with a distribution of data that still resembles the 80-20 ratio
print(f"Training hourly-rate: {(time_train * 100) / (time_train + time_test)}%")
print(f"Test hourly-rate: {(time_test * 100) / (time_train + time_test)}%")
```

### Sample Weighting

#### **Distance Function**

To improve our model, we'll prioritize training samples based on trajectory straightness. We'll compute the geographical distance between a segment's start and end points using the `Haversine` formula. Comparing this to the total distance of all consecutive points will give a straightness metric. Our model will focus on complex trajectories with multiple direction changes, leading to better generalization and more accurate predictions.

```python
def haversine_distance(lon_1, lat_1, lon_2, lat_2):
    lon_1, lat_1, lon_2, lat_2 = map(np.radians, [lon_1, lat_1, lon_2, lat_2])  # convert latitude and longitude to radians
    a = np.sin((lat_2 - lat_1) / 2) ** 2 + np.cos(lat_1) * np.cos(lat_2) * np.sin((lon_2 - lon_1) / 2) ** 2
    return (2 * np.arcsin(np.sqrt(a))) * 6371000  # R: 6,371,000 meters
```

#### **Complexity Score**

_Trajectory straightness calculation using the Haversine:_

```python
def trajectory_straightness(x):
    start_point, end_point = x[0, :2], x[-1, :2]
    x_coordinates, y_coordinates = x[:-1, 0], x[:-1, 1]
    x_coordinates_next, y_coordinates_next = x[1:, 0], x[1:, 1]
    consecutive_distances = np.array(haversine_distance(x_coordinates, y_coordinates, x_coordinates_next, y_coordinates_next))
    straight_line_distance = np.array(haversine_distance(start_point[0], start_point[1], end_point[0], end_point[1]))
    result = straight_line_distance / np.sum(consecutive_distances)
    return result if not np.isnan(result) else 1
```

#### Sample Windowing

To predict **96 data points** (output) using the preceding **48 data points** (input) in a trajectory time series, we create a sliding window. First, we select the initial 48 data points as the input sequence and the subsequent 96 as the output sequence. We then slide the window forward by one step and repeat the process. This continues until the end of the sequence, helping our model capture temporal dependencies and patterns in the data.

Our training strategy uses the sliding window technique, requiring unique weights for each sample. **Sliding Windows (SW)** transforms time series data into an appropriate format for machine learning. They generate overlapping windows with a **fixed number** of consecutive points by sliding the window one step at a time through the series.

```python
def process_voyage(voyage, mmsi, max_size, overlap_size=1):
    straightness_ratios, mmsis, x, y = [], [], [], []

    for j in range(0, voyage.shape[0] - max_size, 1):
        x_sample = voyage[(0 + j):(INPUT_TIMESTEPS + j)]
        y_sample = voyage[(INPUT_TIMESTEPS + j - overlap_size):(max_size + j), 0:OUTPUT_VARIABLES]
        straightness = trajectory_straightness(x_sample)
        straightness_ratios.append(straightness)
        x.append(x_sample.T)
        y.append(y_sample.T)
        mmsis.append(mmsi)

    return straightness_ratios, mmsis, x, y
```

```python
def process_data(voyages):
    max_size = INPUT_TIMESTEPS + OUTPUT_TIMESTEPS

    # Callback function to update tqdm progress bar
    def process_voyage_callback(result, pbar):
        pbar.update(1)
        return result

    with Pool(NUM_WORKERS) as pool, tqdm(total=sum(len(v) for v in voyages.values()), desc="Voyages") as pbar:
        results = []

        # Submit tasks to the pool and store the results
        for mmsi in voyages:
            for voyage in voyages[mmsi]:
                callback = partial(process_voyage_callback, pbar=pbar)
                results.append(pool.apply_async(process_voyage, (voyage, mmsi, max_size), callback=callback))

        pool.close()
        pool.join()

        # Gather the results
        straightness_ratios, mmsis, x, y = [], [], [], []
        for result in results:
            s_ratios, s_mmsis, s_x, s_y = result.get()
            straightness_ratios.extend(s_ratios)
            mmsis.extend(s_mmsis)
            x.extend(s_x)
            y.extend(s_y)

    # Process the results
    x, y = np.stack(x), np.stack(y)
    x, y = np.transpose(x, (0, 2, 1)), np.transpose(y, (0, 2, 1))
    straightness_ratios = np.array(straightness_ratios)
    min_straightness, max_straightness = np.min(straightness_ratios), np.max(straightness_ratios)
    scaled_straightness_ratios = (straightness_ratios - min_straightness) / (max_straightness - min_straightness)
    scaled_straightness_ratios = 1. - scaled_straightness_ratios

    print(f"Final number of samples = {len(x)}", end="\n\n")
    return mmsis, x, y, scaled_straightness_ratios
mmsi_train, x_train, y_train, straightness_ratios = process_data(train_voyage)
mmsi_test, x_test, y_test, _ = process_data(test_voyage)
```

In this project, the input data includes four features: Longitude, Latitude, COG (Course over Ground), and SOG (Speed over Ground), while the output data includes only Longitude and Latitude. To enhance the model's learning, we need to normalize the data through three main steps.

**First,** normalize Longitude, Latitude, COG, and SOG to the \[0, 1] range using domain-specific parameters. This ensures the model performs well in Atlantic Canada waters by restricting the geographical scope of the AIS data and maintaining a similar scale for all features.

**Second,** the input and output data are standardized by subtracting the mean and dividing by the standard deviation. This centers the data around zero and scales it by its variance, preventing vanishing gradients during training.

**Finally,** another zero-one normalization is applied to scale the data to the \[0, 1] range, aligning it with the expected range for many neural network activation functions.

```python
def normalize_dataset(x_train, x_test, y_train,
                      lat_min=42, lat_max=52, lon_min=-70, lon_max=-50, max_sog=50):

    def normalize(arr, min_val, max_val):
        return (arr - min_val) / (max_val - min_val)

    # Initial normalization
    x_train[:, :, :2] = normalize(x_train[:, :, :2], np.array([lon_min, lat_min]), np.array([lon_max, lat_max]))
    y_train[:, :, :2] = normalize(y_train[:, :, :2], np.array([lon_min, lat_min]), np.array([lon_max, lat_max]))
    x_test[:, :, :2] = normalize(x_test[:, :, :2], np.array([lon_min, lat_min]), np.array([lon_max, lat_max]))

    x_train[:, :, 2:4] = x_train[:, :, 2:4] / np.array([360, max_sog])
    x_test[:, :, 2:4] = x_test[:, :, 2:4] / np.array([360, max_sog])

    # Standardize X and Y
    x_mean, x_std = np.mean(x_train, axis=(0, 1)), np.std(x_train, axis=(0, 1))
    y_mean, y_std = np.mean(y_train, axis=(0, 1)), np.std(y_train, axis=(0, 1))

    x_train = (x_train - x_mean) / x_std
    y_train = (y_train - y_mean) / y_std
    x_test = (x_test - x_mean) / x_std

    # Final zero-one normalization
    x_min, x_max = np.min(x_train, axis=(0, 1)), np.max(x_train, axis=(0, 1))
    y_min, y_max = np.min(y_train, axis=(0, 1)), np.max(y_train, axis=(0, 1))

    x_train = (x_train - x_min) / (x_max - x_min)
    y_train = (y_train - y_min) / (y_max - y_min)
    x_test = (x_test - x_min) / (x_max - x_min)

    return x_train, x_test, y_train, y_mean, y_std, y_min, y_max, x_mean, x_std, x_min, x_max
    return x_train, x_test, y_train, y_mean, y_std, y_min, y_max, x_mean, x_std, x_min, x_max

x_train, x_test, y_train, y_mean, y_std, y_min, y_max, x_mean, x_std, x_min, x_max = normalize_dataset(x_train, x_test, y_train)
```

_Denormalizing Y output to the original scale of the data:_

```python
def denormalize_y(y_data, y_mean, y_std, y_min, y_max,
                  lat_min=42, lat_max=52, lon_min=-70, lon_max=-50):
    y_data = y_data * (y_max - y_min) + y_min  # reverse zero-one normalization
    y_data = y_data * y_std + y_mean  # reverse standardization
    # Reverse initial normalization for longitude and latitude
    y_data[:, :, 0] = y_data[:, :, 0] * (lon_max - lon_min) + lon_min
    y_data[:, :, 1] = y_data[:, :, 1] * (lat_max - lat_min) + lat_min
    return y_data
```

_Denormalizing X output to the original scale of the data:_

```python
def denormalize_x(x_data, x_mean, x_std, x_min, x_max,
                  lat_min=42, lat_max=52, lon_min=-70, lon_max=-50):
    x_data = x_data * (x_max - x_min) + x_min  # reverse zero-one normalization
    x_data = x_data * x_std + x_mean  # reverse standardization
    # Reverse initial normalization for longitude and latitude
    x_data[:, :, 0] = x_data[:, :, 0] * (lon_max - lon_min) + lon_min
    x_data[:, :, 1] = x_data[:, :, 1] * (lat_max - lat_min) + lat_min
    return x_data
```

machine-learningWe have successfully prepared the data for our machine-learning task. With the data ready, it's time for the modeling phase. Next, we will create, train, and evaluate a machine-learning model to forecast vessel trajectories using the processed dataset. Let's explore how our model performs in Atlantic Canada!

```python
tf.keras.backend.clear_session()  # Clear the Keras session to prevent potential conflicts
_ = wandb.login(force=True)  # Log in to Weights & Biases
```

## Gated Recurrent Unit AutoEncoder

A GRU Autoencoder is a neural network that compresses and reconstructs sequential data utilizing a **Gated Recurrent Unit**. _GRUs are highly effective at handling time-series data, which are sequential data points captured over time, as they can model intricate temporal dependencies and patterns._ To perform time-series forecasting, a GRU Autoencoder can be trained on a historical time-series dataset to discern patterns and trends, subsequently compressing a sequence of future data points into a lower-dimensional representation that can be decoded to generate a forecast of the upcoming data points. With this in mind, we will begin by constructing a model architecture composed of two GRU layers with **64 units each**, taking input of shape **(48, 4)** and **(96, 4)**, respectively, followed by a dense layer with **2 units**.

```python
class ProbabilisticTeacherForcing(Layer):
    def __init__(self, **kwargs):
        super(ProbabilisticTeacherForcing, self).__init__(**kwargs)

    def call(self, inputs):
        decoder_gt_input, decoder_output, mixing_prob = inputs
        mixing_prob = tf.expand_dims(mixing_prob, axis=-1)  # Add an extra dimension for broadcasting
        mixing_prob = tf.broadcast_to(mixing_prob, tf.shape(decoder_gt_input))  # Broadcast to match the shape
        return tf.where(tf.random.uniform(tf.shape(decoder_gt_input)) < mixing_prob, decoder_gt_input, decoder_output)
```

```python
def build_model(rnn_unit="GRU", hidden_size=64):
    encoder_input = Input(shape=(INPUT_TIMESTEPS, INPUT_VARIABLES), name="Encoder_Input")
    decoder_gt_input = Input(shape=((OUTPUT_TIMESTEPS - 1), OUTPUT_VARIABLES), name="Decoder-GT-Input")
    mixing_prob_input = Input(shape=(1,), name="Mixing_Probability")

    # Encoder
    encoder_gru = eval(rnn_unit)(hidden_size, activation="relu", name="Encoder")(encoder_input)
    repeat_vector = RepeatVector((OUTPUT_TIMESTEPS - 1), name="Repeater")(encoder_gru)

    # Inference Decoder
    decoder_gru = eval(rnn_unit)(hidden_size, activation="relu", return_sequences=True, name="Decoder")
    decoder_output = decoder_gru(repeat_vector, initial_state=encoder_gru)

    # Adjust decoder_output shape
    dense_output_adjust = TimeDistributed(Dense(OUTPUT_VARIABLES), name="Output_Adjust")
    adjusted_decoder_output = dense_output_adjust(decoder_output)

    # Training Decoder
    decoder_gru_tf = eval(rnn_unit)(hidden_size, activation="relu", return_sequences=True, name="Decoder-TF")
    probabilistic_tf_layer = ProbabilisticTeacherForcing(name="Probabilistic_Teacher_Forcing")
    mixed_input = probabilistic_tf_layer([decoder_gt_input, adjusted_decoder_output, mixing_prob_input])
    tf_output = decoder_gru_tf(mixed_input, initial_state=encoder_gru)
    tf_output = dense_output_adjust(tf_output)  # Use dense_output_adjust layer for training output

    training_model = Model(inputs=[encoder_input, decoder_gt_input, mixing_prob_input], outputs=tf_output, name="Training")
    inference_model = Model(inputs=encoder_input, outputs=adjusted_decoder_output, name="Inference")

    return training_model, inference_model
training_model, model = build_model()
```

### **Custom Loss Function**

```python
def denormalize_y(y_data, y_mean, y_std, y_min, y_max, lat_min=42, lat_max=52, lon_min=-70, lon_max=-50):
    scales = tf.constant([lon_max - lon_min, lat_max - lat_min], dtype=tf.float32)
    biases = tf.constant([lon_min, lat_min], dtype=tf.float32)

    # Reverse zero-one normalization and standardization
    y_data = y_data * (y_max - y_min) + y_min
    y_data = y_data * y_std + y_mean

    # Reverse initial normalization for longitude and latitude
    return y_data * scales + biases
```

```python
def haversine_distance(lon1, lat1, lon2, lat2):
    lon1, lat1, lon2, lat2 = [tf.math.multiply(x, tf.divide(tf.constant(np.pi), 180.)) for x in [lon1, lat1, lon2, lat2]]  # lat and lon to radians
    a = tf.math.square(tf.math.sin((lat2 - lat1) / 2.)) + tf.math.cos(lat1) * tf.math.cos(lat2) * tf.math.square(tf.math.sin((lon2 - lon1) / 2.))
    return 2 * 6371000 * tf.math.asin(tf.math.sqrt(a))  # The earth Radius is 6,371,000 meters
```

```python
def custom_loss(y_true, y_pred):
    tf.debugging.check_numerics(y_true, "y_true contains NaNs")
    tf.debugging.check_numerics(y_pred, "y_pred contains NaNs")

    # Denormalize true and predicted y
    y_true_denorm = denormalize_y(y_true, y_mean, y_std, y_min, y_max)
    y_pred_denorm = denormalize_y(y_pred, y_mean, y_std, y_min, y_max)

    # Compute haversine distance for true and predicted y from the second time-step
    true_dist = haversine_distance(y_true_denorm[:, 1:, 0], y_true_denorm[:, 1:, 1], y_true_denorm[:, :-1, 0], y_true_denorm[:, :-1, 1])
    pred_dist = haversine_distance(y_pred_denorm[:, 1:, 0], y_pred_denorm[:, 1:, 1], y_pred_denorm[:, :-1, 0], y_pred_denorm[:, :-1, 1])

    # Convert maximum speed from knots to meters per 5 minutes
    max_speed_m_per_5min = 50 * 1.852 * 1000 * 5 / 60

    # Compute the difference in distances
    dist_diff = tf.abs(true_dist - pred_dist)

    # Apply penalty if the predicted distance is greater than the maximum possible distance
    dist_diff = tf.where(pred_dist > max_speed_m_per_5min, pred_dist - max_speed_m_per_5min, dist_diff)

    # Penalty for the first output coordinate not being the same as the last input
    input_output_diff = haversine_distance(y_true_denorm[:, 0, 0], y_true_denorm[:, 0, 1], y_pred_denorm[:, 0, 0], y_pred_denorm[:, 0, 1])

    # Compute RMSE excluding the first element
    rmse = K.sqrt(K.mean(K.square(y_true_denorm[:, 1:, :] - y_pred_denorm[:, 1:, :]), axis=1))

    tf.debugging.check_numerics(y_true_denorm, "y_true_denorm contains NaNs")
    tf.debugging.check_numerics(y_pred_denorm, "y_pred_denorm contains NaNs")
    tf.debugging.check_numerics(true_dist, "true_dist contains NaNs")
    tf.debugging.check_numerics(pred_dist, "pred_dist contains NaNs")
    tf.debugging.check_numerics(dist_diff, "dist_diff contains NaNs")
    tf.debugging.check_numerics(input_output_diff, "input_output_diff contains NaNs")
    tf.debugging.check_numerics(rmse, "rmse contains NaNs")

    # Final loss with weights
    # return 0.25 * K.mean(input_output_diff) + 0.35 * K.mean(dist_diff) + 0.40 * K.mean(rmse)
    return K.mean(rmse)
```

### **Model Summary**

```python
def compile_model(model, learning_rate, clipnorm, jit_compile, skip_summary=False):
    optimizer = AdamW(learning_rate=learning_rate, clipnorm=clipnorm, jit_compile=jit_compile)
    model.compile(optimizer=optimizer, loss=custom_loss, metrics=["mae", "mape"], weighted_metrics=[], jit_compile=jit_compile)
    if not skip_summary: model.summary()  # print a summary of the model architecture
compile_model(training_model, learning_rate=0.001, clipnorm=1, jit_compile=True)
compile_model(model, learning_rate=0.001, clipnorm=1, jit_compile=True)
```

{% code fullWidth="true" %}
```brightscript
Model: "Training"
__________________________________________________________________________________________________
 Layer (type)                   Output Shape         Param #     Connected to                     
==================================================================================================
 Encoder_Input (InputLayer)     [(None, 48, 8)]      0           []                               
                                                                                                  
 Encoder (GRU)                  (None, 64)           14208       ['Encoder_Input[0][0]']          
                                                                                                  
 Repeater (RepeatVector)        (None, 95, 64)       0           ['Encoder[0][0]']                
                                                                                                  
 Decoder (GRU)                  (None, 95, 64)       24960       ['Repeater[0][0]',               
                                                                  'Encoder[0][0]']                
                                                                                                  
 Output_Adjust (TimeDistributed  (None, 95, 2)       130         ['Decoder[0][0]',                
 )                                                                'Decoder-TF[0][0]']             
                                                                                                  
 Decoder-GT-Input (InputLayer)  [(None, 95, 2)]      0           []                               
                                                                                                  
 Mixing_Probability (InputLayer  [(None, 1)]         0           []                               
 )                                                                                                
                                                                                                  
 Probabilistic_Teacher_Forcing   (None, 95, 2)       0           ['Decoder-GT-Input[0][0]',       
 (ProbabilisticTeacherForcing)                                    'Output_Adjust[0][0]',          
                                                                  'Mixing_Probability[0][0]']     
                                                                                                  
 Decoder-TF (GRU)               (None, 95, 64)       13056       ['Probabilistic_Teacher_Forcing[0
                                                                 ][0]',                           
                                                                  'Encoder[0][0]']                
                                                                                                  
==================================================================================================
Total params: 52,354
Trainable params: 52,354
Non-trainable params: 0
__________________________________________________________________________________________________
Model: "Inference"
__________________________________________________________________________________________________
 Layer (type)                   Output Shape         Param #     Connected to                     
==================================================================================================
 Encoder_Input (InputLayer)     [(None, 48, 8)]      0           []                               
                                                                                                  
 Encoder (GRU)                  (None, 64)           14208       ['Encoder_Input[0][0]']          
                                                                                                  
 Repeater (RepeatVector)        (None, 95, 64)       0           ['Encoder[0][0]']                
                                                                                                  
 Decoder (GRU)                  (None, 95, 64)       24960       ['Repeater[0][0]',               
                                                                  'Encoder[0][0]']                
                                                                                                  
 Output_Adjust (TimeDistributed  (None, 95, 2)       130         ['Decoder[0][0]']                
 )                                                                                                
                                                                                                  
==================================================================================================
Total params: 39,298
Trainable params: 39,298
Non-trainable params: 0
__________________________________________________________________________________________________
```
{% endcode %}

### Training Callbacks

The following function lists callbacks used during the model training process. Callbacks are utilities at specific points during training to monitor progress or take actions based on the model's performance. The function pre-define the parameters and behavior of these callbacks:

* **WandbMetricsLogger:** This callback logs the training and validation metrics for visualization and monitoring on the Weights & Biases (W\&B) platform. This can be useful for tracking the training progress but may introduce additional overhead due to the logging process. You can remove this callback if you don't need to use W\&B or want to reduce the overhead.
* **TerminateOnNaN:** This callback terminates training if the loss becomes `NaN` (Not a Number) during the training process. It helps to stop the training process early when the model diverges and encounters an unstable state.
* **ReduceLROnPlateau:** This callback reduces the learning rate by a specified factor when the monitored metric has stopped improving for several epochs. It helps fine-tune the model using a lower learning rate when it no longer improves significantly.
* **EarlyStopping:** This callback stops the training process early when the monitored metric has not improved for a specified number of epochs. It restores the model's best weights when the training is terminated, preventing overfitting and reducing the training time.
* **ModelCheckpoint:** This callback saves the best model (based on the monitored metric) to a file during training.

`WandbMetricsLogger` is the most computationally costly among these callbacks due to the logging process. You can remove this callback if you don't need to use Weights & Biases for monitoring or want to reduce overhead. The other callbacks help optimize the training process and are less computationally demanding. It's important to note that the **Weights & Biases (W\&B)** platform is also used in other parts of the code. If you decide to remove the `WandbMetricsLogger` callback, please ensure that you also remove any other references to W\&B in the code to avoid potential issues. If you choose to use W\&B for monitoring and logging, you must register and log in to the [W\&B website](https://wandb.ai/). During the execution of the code, you'll be prompted for an authentication key to connect your script to **your W\&B account**. _This key can be obtained from your W\&B account settings._ Once you have the key, you can use it to enable W\&B's monitoring and logging features provided by W\&B.

```python
def create_callbacks(model_name, monitor="val_loss", factor=0.2, lr_patience=3, ep_patience=12, min_lr=0, verbose=0, restore_best_weights=True, skip_wandb=False):
    return  ([wandb.keras.WandbMetricsLogger()] if not skip_wandb else []) + [#tf.keras.callbacks.TerminateOnNaN(),
            ReduceLROnPlateau(monitor=monitor, factor=factor, patience=lr_patience, min_lr=min_lr, verbose=verbose),
            EarlyStopping(monitor=monitor, patience=ep_patience, verbose=verbose, restore_best_weights=restore_best_weights),
            tf.keras.callbacks.ModelCheckpoint(os.path.join(ROOT, MODELS, model_name), monitor="val_loss", mode="min", save_best_only=True, verbose=verbose)]
```

### **Model Training**

```python
def train_model(model, x_train, y_train, batch_size, epochs, validation_split, model_name):
    run = wandb.init(project="kAISdb", anonymous="allow")  # start the wandb run

    # Set the initial mixing probability
    mixing_prob = 0.5

    # Update y_train to have the same dimensions as the output
    y_train = y_train[:, :(OUTPUT_TIMESTEPS - 1), :]

    # Create the ground truth input for the decoder by appending a padding at the beginning of the sequence
    decoder_ground_truth_input_data = (np.zeros((y_train.shape[0], 1, y_train.shape[2])), y_train[:, :-1, :])
    decoder_ground_truth_input_data = np.concatenate(decoder_ground_truth_input_data, axis=1)

    try:
        # Train the model with Teacher Forcing
        with tf.device(tf.test.gpu_device_name()):
            training_model.fit([x_train, decoder_ground_truth_input_data, np.full((x_train.shape[0], 1), mixing_prob)], y_train, batch_size=batch_size, epochs=epochs,
                            verbose=2, validation_split=validation_split, callbacks=create_callbacks(model_name))
            # , sample_weight=straightness_ratios)
    except KeyboardInterrupt as e:
        print("\nRestoring best weights [...]")
        # Load the weights of the teacher-forcing model
        training_model.load_weights(model_name)
        # Transfering the weights to the inference model
        for layer in model.layers:
            if layer.name in [l.name for l in training_model.layers]:
                layer.set_weights(training_model.get_layer(layer.name).get_weights())

    run.finish()  # finish the wandb run
model_name = "TF-GRU-AE.h5"
full_path = os.path.join(ROOT, MODELS, model_name)
if True:#not os.path.exists(full_path):
    train_model(model, x_train, y_train, batch_size=1024,
                epochs=250, validation_split=0.2,
                model_name=model_name)
else:
    training_model.load_weights(full_path)
    for layer in model.layers:  # inference model initialization
        if layer.name in [l.name for l in training_model.layers]:
            layer.set_weights(training_model.get_layer(layer.name).get_weights())
```

### **Model Evaluation**

```python
def evaluate_model(model, x_test, y_test, y_mean, y_std, y_min, y_max, y_pred=None):

    def single_trajectory_error(y_test, y_pred, index):
        distances = haversine_distance(y_test[index, :, 0], y_test[index, :, 1], y_pred[index, :, 0], y_pred[index, :, 1])
        return np.min(distances), np.max(distances), np.mean(distances), np.median(distances)

    # Modify this function to handle teacher-forced models with 95 output variables instead of 96
    def all_trajectory_error(y_test, y_pred):
        errors = [single_trajectory_error(y_test[:, 1:], y_pred, i) for i in range(y_test.shape[0])]
        min_errors, max_errors, mean_errors, median_errors = zip(*errors)
        return min(min_errors), max(max_errors), np.mean(mean_errors), np.median(median_errors)

    def plot_trajectory(x_test, y_test, y_pred, sample_index):
        min_error, max_error, mean_error, median_error = single_trajectory_error(y_test, y_pred, sample_index)
        fig = go.Figure()

        fig.add_trace(go.Scatter(x=x_test[sample_index, :, 0], y=x_test[sample_index, :, 1], mode="lines", name="Input Data", line=dict(color="green")))
        fig.add_trace(go.Scatter(x=y_test[sample_index, :, 0], y=y_test[sample_index, :, 1], mode="lines", name="Ground Truth", line=dict(color="blue")))
        fig.add_trace(go.Scatter(x=y_pred[sample_index, :, 0], y=y_pred[sample_index, :, 1], mode="lines", name="Forecasted Trajectory", line=dict(color="red")))

        fig.update_layout(title=f"Sample Index: {sample_index} | Distance Errors (in meteres):<br>Min: {min_error:.2f}m, Max: {max_error:.2f}m, "
                                f"Mean: {mean_error:.2f}m, Median: {median_error:.2f}m", xaxis_title="Longitude", yaxis_title="Latitude",
                          plot_bgcolor="#e4eaf0", paper_bgcolor="#fcfcfc", width=700, height=600)

        max_lon, max_lat = -58.705587131108196, 47.89066160591873
        min_lon, min_lat = -61.34247286889181, 46.09201839408127

        fig.update_xaxes(range=[min_lon, max_lon])
        fig.update_yaxes(range=[min_lat, max_lat])

        return fig

    if y_pred is None:
        with tf.device(tf.test.gpu_device_name()):
            y_pred = model.predict(x_test, verbose=0)
    y_pred_o = y_pred  # preserve the result

    x_test = denormalize_x(x_test, x_mean, x_std, x_min, x_max)
    y_pred = denormalize_y(y_pred_o, y_mean, y_std, y_min, y_max)

    # Modify this line to handle teacher-forced models with 95 output variables instead of 96
    for sample_index in [1000, 2500, 5000, 7500]:
        display(plot_trajectory(x_test, y_test[:, 1:], y_pred, sample_index))

    # The metrics require a lower dimension (no impact on the results)
    y_test_reshaped = np.reshape(y_test[:, 1:], (-1, y_test.shape[2]))
    y_pred_reshaped = np.reshape(y_pred, (-1, y_pred.shape[2]))

    # Physical Distance Error given in meters
    all_min_error, all_max_error, all_mean_error, all_median_error = all_trajectory_error(y_test, y_pred)

    print("\nAll Trajectories Min DE: {:.4f}m".format(all_min_error))
    print("All Trajectories Max DE: {:.4f}m".format(all_max_error))
    print("All Trajectories Mean DE: {:.4f}m".format(all_mean_error))
    print("All Trajectories Median DE: {:.4f}m".format(all_median_error))

    # Calculate evaluation metrics on the test data
    r2 = r2_score(y_test_reshaped, y_pred_reshaped)
    mse = mean_squared_error(y_test_reshaped, y_pred_reshaped)
    mae = mean_absolute_error(y_test_reshaped, y_pred_reshaped)
    evs = explained_variance_score(y_test_reshaped, y_pred_reshaped)
    mape = mean_absolute_percentage_error(y_test_reshaped, y_pred_reshaped)
    rmse = np.sqrt(mse)

    print(f"\nTest R^2: {r2:.4f}")
    print(f"Test MAE: {mae:.4f}")
    print(f"Test MSE: {mse:.4f}")
    print(f"Test RMSE: {rmse:.4f}")
    print(f"Test MAPE: {mape:.4f}")
    print(f"Test Explained Variance Score: {evs:.4f}")

    return y_pred_o
_ = evaluate_model(model, x_test, y_test, y_mean, y_std, y_min, y_max)
```

## Hyperparameters Tuning

In this step, we define a function called `model_placeholder` that uses the **Keras Tuner** to create a model with tunable hyperparameters. The function takes a hyperparameter object as input, which defines the search space for the hyperparameters of interest. Specifically, we are searching for the best number of units in the encoder and decoder GRU layers and the optimal learning rate for the AdamW optimizer. The `model_placeholder` function constructs a GRU-AutoEncoder model with these tunable hyperparameters and compiles the model using the Mean Absolute Error (MAE) as the loss function. **Keras Tuner** will use this model during the hyperparameter optimization process to find the best combination of hyperparameters that minimizes the validation loss at the expanse of long computing time.

_Helper for saving the training history:_

```python
def save_history(history, model_name):
    history_name = model_name.replace('.h5', '.pkl')
    history_name = os.path.join(ROOT, MODELS, history_name)
    with open(history_name, 'wb') as f:
        pkl.dump(history, f)
```

_Helper for restoring the training history:_

```python
def load_history(model_name):
    history_name = model_name.replace('.h5', '.pkl')
    history_name = os.path.join(ROOT, MODELS, history_name)
    with open(history_name, 'rb') as f:
        history = pkl.load(f)
    return history
```

_Defining the model to be optimized:_

```python
def build_model(rnn_unit="GRU", enc_units_1=64, dec_units_1=64):
    encoder_input = Input(shape=(INPUT_TIMESTEPS, INPUT_VARIABLES), name="Encoder_Input")
    decoder_gt_input = Input(shape=((OUTPUT_TIMESTEPS - 1), OUTPUT_VARIABLES), name="Decoder-GT-Input")
    mixing_prob_input = Input(shape=(1,), name="Mixing_Probability")

    # Encoder
    encoder_gru = eval(rnn_unit)(enc_units_1, activation="relu", name="Encoder")(encoder_input)
    repeat_vector = RepeatVector((OUTPUT_TIMESTEPS - 1), name="Repeater")(encoder_gru)

    # Inference Decoder
    decoder_gru = eval(rnn_unit)(dec_units_1, activation="relu", return_sequences=True, name="Decoder")
    decoder_output = decoder_gru(repeat_vector, initial_state=encoder_gru)

    # Adjust decoder_output shape
    dense_output_adjust = TimeDistributed(Dense(OUTPUT_VARIABLES), name="Output_Adjust")
    adjusted_decoder_output = dense_output_adjust(decoder_output)

    # Training Decoder
    decoder_gru_tf = eval(rnn_unit)(dec_units_1, activation="relu", return_sequences=True, name="Decoder-TF")
    probabilistic_tf_layer = ProbabilisticTeacherForcing(name="Probabilistic_Teacher_Forcing")
    mixed_input = probabilistic_tf_layer([decoder_gt_input, adjusted_decoder_output, mixing_prob_input])
    tf_output = decoder_gru_tf(mixed_input, initial_state=encoder_gru)
    tf_output = dense_output_adjust(tf_output)  # Use dense_output_adjust layer for training output

    training_model = Model(inputs=[encoder_input, decoder_gt_input, mixing_prob_input], outputs=tf_output, name="Training")
    inference_model = Model(inputs=encoder_input, outputs=adjusted_decoder_output, name="Inference")

    return training_model, inference_model
```

_HyperOpt Objective Function:_

```python
def objective(hyperparams, x_train, y_train, straightness_ratios, model_prefix):
    # Get the best hyperparameters from the optimization results
    enc_units_1 = hyperparams["enc_units_1"]
    dec_units_1 = hyperparams["dec_units_1"]
    mixing_prob = hyperparams["mixing_prob"]
    lr = hyperparams["learning_rate"]

    # Create the model name using the best hyperparameters
    model_name = f"{model_prefix}-{enc_units_1}-{dec_units_1}-{mixing_prob}-{lr}.h5"
    full_path = os.path.join(ROOT, MODELS, model_name)  # best model full path

    # Check if the model results file with this name already exists
    if not os.path.exists(full_path.replace(".h5", ".pkl")):
        print(f"Saving under {model_name}.")

        # Define the model architecture
        training_model, _ = build_model(enc_units_1=enc_units_1, dec_units_1=dec_units_1)
        compile_model(training_model, learning_rate=lr, clipnorm=1, jit_compile=True, skip_summary=True)

        # Update y_train to have the same dimensions as the output
        y_train = y_train[:, :(OUTPUT_TIMESTEPS - 1), :]

        # Create the ground truth input for the decoder by appending a padding at the beginning of the sequence
        decoder_ground_truth_input_data = (np.zeros((y_train.shape[0], 1, y_train.shape[2])), y_train[:, :-1, :])
        decoder_ground_truth_input_data = np.concatenate(decoder_ground_truth_input_data, axis=1)

        # Train the model on the data, using GPU if available
        with tf.device(tf.test.gpu_device_name()):
            history = training_model.fit([x_train, decoder_ground_truth_input_data, np.full((x_train.shape[0], 1), mixing_prob)], y_train,
                                         batch_size=10240, epochs=250, validation_split=.2, verbose=0,
                                         workers=multiprocessing.cpu_count(), use_multiprocessing=True,
                                         callbacks=create_callbacks(model_name, skip_wandb=True))
                                         #, sample_weight=straightness_ratios)
            # Save the training history
            save_history(history.history, model_name)

        # Clear the session to release resources
        del training_model; tf.keras.backend.clear_session()
    else:
        print("Loading pre-trained weights.")
        history = load_history(model_name)

    if type(history) == dict:  # validation loss of the model
        return {"loss": history["val_loss"][-1], "status": STATUS_OK}
    else: return {"loss": history.history["val_loss"][-1], "status": STATUS_OK}
```

### **Search for Best Model**

```python
def optimize_hyperparameters(max_evals, model_prefix, x_train, y_train, sample_size=5000):

    def build_space(n_min=2, n_steps=9):
        # Defining a custom 2^N range function
        n_range = lambda n_min, n_steps: np.array(
            [2**n for n in range(n_min, n_steps) if 2**n >= n_min])

        # Defining the unconstrained search space
        encoder_1_range = n_range(n_min, n_steps)
        decoder_1_range = n_range(n_min, n_steps)
        learning_rate_range = [.01, .001, .0001]
        mixing_prob_range = [.25, .5, .75]

        # Enforcinf contraints to the search space
        enc_units_1 = np.random.choice(encoder_1_range)
        dec_units_1 = np.random.choice(decoder_1_range[np.where(decoder_1_range == enc_units_1)])
        learning_rate = np.random.choice(learning_rate_range)
        mixing_prob = np.random.choice(mixing_prob_range)

        # Returns a single element of the search space
        return dict(enc_units_1=enc_units_1, dec_units_1=dec_units_1, learning_rate=learning_rate, mixing_prob=mixing_prob)

    # Select the search space based on a pre-set sampled random space
    search_space = hp.choice("hyperparams", [build_space() for _ in range(sample_size)])
    trials = Trials()  # initialize Hyperopt trials

    # Define the objective function for Hyperopt
    fn = lambda hyperparams: objective(hyperparams, x_train, y_train, straightness_ratios, model_prefix)

    # Perform Hyperopt optimization and find the best hyperparameters
    best = fmin(fn=fn, space=search_space, algo=tpe.suggest, max_evals=max_evals, trials=trials)
    best_hyperparams = space_eval(search_space, best)

    # Get the best hyperparameters from the optimization results
    enc_units_1 = best_hyperparams["enc_units_1"]
    dec_units_1 = best_hyperparams["dec_units_1"]
    mixing_prob = best_hyperparams["mixing_prob"]
    lr = best_hyperparams["learning_rate"]

    # Create the model name using the best hyperparameters
    model_name = f"{model_prefix}-{enc_units_1}-{dec_units_1}-{mixing_prob}-{lr}.h5"
    full_path = os.path.join(ROOT, MODELS, model_name)  # best model full path
    t_model, i_model = build_model(enc_units_1=enc_units_1, dec_units_1=dec_units_1)
    t_model = tf.keras.models.load_model(full_path)
    for layer in i_model.layers:  # inference model initialization
        if layer.name in [l.name for l in t_model.layers]:
            layer.set_weights(t_model.get_layer(layer.name).get_weights())

    print(f"Best hyperparameters:")
    print(f"  Encoder units 1: {enc_units_1}")
    print(f"  Decoder units 1: {dec_units_1}")
    print(f"  Mixing proba.: {mixing_prob}")
    print(f"  Learning rate: {lr}")
    return i_model
max_evals, model_prefix = 100, "TF-GRU"
# best_model = optimize_hyperparameters(max_evals, model_prefix, x_train, y_train)
# [NOTE] YOU CAN SKIP THIS STEP BY LOADING THE PRE-TRAINED WEIGHTS ON THE NEXT CELL.
```

_Swiping the project folder for other pre-trained weights shared with this tutorial:_

```python
def find_best_model(root_folder, model_prefix):
    best_model_name, best_val_loss = None, float('inf')

    for f in os.listdir(root_folder):
        if (f.endswith(".h5") and f.startswith(model_prefix)):
            try:
                history = load_history(f)

                # Get the validation loss
                if type(history) == dict:
                    val_loss = history["val_loss"][-1]
                else: val_loss = history.history["val_loss"][-1]

                # Storing the best model
                if val_loss < best_val_loss:
                    best_val_loss = val_loss
                    best_model_name = f
            except: pass

    # Load the best model
    full_path = os.path.join(ROOT, MODELS, best_model_name)
    t_model, i_model = build_model(enc_units_1=int(best_model_name.split("-")[2]), dec_units_1=int(best_model_name.split("-")[3]))
    t_model = tf.keras.models.load_model(full_path, custom_objects={"ProbabilisticTeacherForcing": ProbabilisticTeacherForcing})
    for layer in i_model.layers:  # inference model initialization
        if layer.name in [l.name for l in t_model.layers]:
            layer.set_weights(t_model.get_layer(layer.name).get_weights())
    # Print summary of the best model
    print(f"Loss: {best_val_loss}")
    i_model.summary()

    return i_model
best_model = find_best_model(os.path.join(ROOT, MODELS), model_prefix)
```

### **Evaluating Best Model**

```python
_ = evaluate_model(best_model, x_test, y_test, y_mean, y_std, y_min, y_max)
```

## Model Explainability

### **Permutation Feature Importance (PFI)**

Deep learning models, although powerful, are often criticized for their lack of explainability, making it difficult to comprehend their decision-making process and raising concerns about trust and reliability. To address this issue, we can use techniques like the PFI method, _a simple, model-agnostic approach that helps visualize the importance of features in deep learning models_. This method works by shuffling individual feature values in the dataset and observing the impact on the model's performance. By measuring the change in a designated metric when each feature's values are randomly permuted, we can infer the importance of that specific feature. The idea is that if a feature is crucial for the model's performance, shuffling its values should lead to a significant shift in performance; otherwise if a feature has little impact, its value permutation should result in a minor change. Applying the permutation feature importance method to the best model, obtained after hyperparameter tuning, can give us a more transparent understanding of how the model makes its decisions.

```python
def permutation_feature_importance(model, x_test, y_test, metric):

    # Function to calculate permutation feature importance
    def PFI(model, x, y_true, metric):
        # Reshape the true values for easier comparison with predictions
        y_true = np.reshape(y_true, (-1, y_true.shape[2]))

        # Predict using the model and reshape the predicted values
        with tf.device(tf.test.gpu_device_name()):
            y_pred = model.predict(x, verbose=0)
        y_pred = np.reshape(y_pred, (-1, y_pred.shape[2]))

        # Calculate the baseline score using the given metric
        baseline_score = metric(y_true, y_pred)

        # Initialize an array for feature importances
        feature_importances = np.zeros(x.shape[2])

        # Calculate the importance for each feature
        for feature_idx in range(x.shape[2]):
            x_permuted = x.copy()
            x_permuted[:, :, feature_idx] = np.random.permutation(x[:, :, feature_idx])

            # Predict using the permuted input and reshape the predicted values
            with tf.device(tf.test.gpu_device_name()):
                y_pred_permuted = model.predict(x_permuted, verbose=0)
            y_pred_permuted = np.reshape(y_pred_permuted, (-1, y_pred_permuted.shape[2]))

            # Calculate the score with permuted input
            permuted_score = metric(y_true, y_pred_permuted)

            # Compute the feature importance as the difference between permuted and baseline scores
            feature_importances[feature_idx] = permuted_score - baseline_score

        return feature_importances

    feature_importances = PFI(model, x_test, y_test, metric)

    # Prepare the data for plotting (require a dataframe)
    feature_names = ["Longitude", "Latitude", "COG", "SOG"]
    feature_importance_df = pd.DataFrame({"features": feature_names, "importance": feature_importances})

    # Create the bar plot with Altair
    bar_plot = alt.Chart(feature_importance_df).mark_bar(size=40, color="mediumblue", opacity=0.8).encode(
        x=alt.X("features:N", title="Features", axis=alt.Axis(labelFontSize=12, titleFontSize=14)),
        y=alt.Y("importance:Q", title="Permutation Importance", axis=alt.Axis(labelFontSize=12, titleFontSize=14)),
    ).properties(title=alt.TitleParams(text="Feature Importance", fontSize=16, fontWeight="bold"), width=400, height=300)

    return bar_plot, feature_importances
permutation_feature_importance(best_model, x_test, y_test, mean_absolute_error)[0].display()
```

### **Sensitivity Analysis**

Permutation feature importance has some limitations, such as assuming features are independent and producing biased results when features are highly correlated. It also doesn't provide detailed explanations for individual data points. An alternative is sensitivity analysis, which studies how input features affect model predictions. By perturbing each input feature individually and observing the prediction changes, we can understand which features significantly impact the model's output. This approach offers insights into the model's decision-making process and helps identify influential features. However, it does not account for feature interactions and can be computationally expensive for many features or perturbation steps.

```python
def sensitivity_analysis(model, x_sample, perturbation_range=(-0.1, 0.1), num_steps=10, plot_nrows=4):
    # Get the number of features and outputs
    num_features = x_sample.shape[1]
    num_outputs = model.output_shape[-1] * model.output_shape[-2]

    # Create an array of perturbations
    perturbations = np.linspace(perturbation_range[0], perturbation_range[1], num_steps)

    # Initialize sensitivity array
    sensitivity = np.zeros((num_features, num_outputs, num_steps))

    # Get the original prediction for the input sample
    original_prediction = model.predict(x_sample.reshape(1, -1, 4), verbose=0).reshape(-1)

    # Iterate over input features and perturbations
    for feature_idx in range(num_features):
        for i, perturbation in enumerate(perturbations):
            # Create a perturbed version of the input sample
            perturbed_sample = x_sample.copy()
            perturbed_sample[:, feature_idx] += perturbation

            # Get the prediction for the perturbed input sample
            perturbed_prediction = model.predict(perturbed_sample.reshape(1, -1, 4), verbose=0).reshape(-1)

            # Calculate the absolute prediction change and store it in the sensitivity array
            sensitivity[feature_idx, :, i] = np.abs(perturbed_prediction - original_prediction)

    # Determine the number of rows and columns in the plot
    ncols = 6
    nrows = max(min(plot_nrows, math.ceil(num_outputs / ncols)), 1)

    # Define feature names
    feature_names = ["Longitude", "Latitude", "COG", "SOG"]

    # Create the sensitivity plot
    fig, axs = plt.subplots(nrows, ncols, figsize=(18, 3 * nrows), sharex=True, sharey=True)
    axs = axs.ravel()

    output_idx = 0
    for row in range(nrows):
        for col in range(ncols):
            if output_idx < num_outputs:
                # Plot sensitivity curves for each feature
                for feature_idx in range(num_features):
                    axs[output_idx].plot(perturbations, sensitivity[feature_idx, output_idx], label=f'{feature_names[feature_idx]}')
                # Set the title for each subplot
                axs[output_idx].set_title(f'Output {output_idx // 2 + 1}, {"Longitude" if output_idx % 2 == 0 else "Latitude"}')
                output_idx += 1

    # Set common labels and legend
    fig.text(0.5, 0.04, 'Perturbation', ha='center', va='center')
    fig.text(0.06, 0.5, 'Absolute Prediction Change', ha='center', va='center', rotation='vertical')
    handles, labels = axs[0].get_legend_handles_labels()
    fig.legend(handles, labels, loc='upper center', ncol=num_features, bbox_to_anchor=(.5, .87))

    plt.tight_layout()
    plt.subplots_adjust(top=0.8, bottom=0.1, left=0.1, right=0.9)
    plt.show()

    return sensitivity
x_sample = x_test[100]  # Select a sample from the test set
sensitivity = sensitivity_analysis(best_model, x_sample)
```

### **UMAP: Uniform Manifold Approximation and Projection**

UMAP is a nonlinear dimensionality reduction technique that visualizes high-dimensional data in a lower-dimensional space, preserving the local and global structure. In trajectory forecasting, UMAP can project high-dimensional model representations into 2D or 3D to clarify the relationships between input features and outputs. Unlike sensitivity analysis, which measures prediction changes due to input feature perturbations, UMAP reveals data structure without perturbations. It also differs from feature permutation, which evaluates feature importance by shuffling values and assessing model performance changes. UMAP focuses on visualizing intrinsic data structures and relationships.

```python
def visualize_intermediate_representations(model, x_test_subset, y_test_subset, n_neighbors=15, min_dist=0.1, n_components=2):
    # Extract intermediate representations from your model
    intermediate_layer_model = keras.Model(inputs=model.input, outputs=model.layers[-2].output)
    intermediate_output = intermediate_layer_model.predict(x_test_subset, verbose=0)

    # Flatten the last two dimensions of the intermediate_output
    flat_intermediate_output = intermediate_output.reshape(intermediate_output.shape[0], -1)

    # UMAP
    reducer = umap.UMAP(n_neighbors=n_neighbors, min_dist=min_dist, n_components=n_components, random_state=seed_value)
    umap_output = reducer.fit_transform(flat_intermediate_output)

    # Convert y_test_subset to strings
    y_test_str = np.array([str(label) for label in y_test_subset])

    # Map string labels to colors
    unique_labels = np.unique(y_test_str)
    colormap = plt.cm.get_cmap('viridis', len(unique_labels))
    label_to_color = {label: colormap(i) for i, label in enumerate(unique_labels)}
    colors = np.array([label_to_color[label] for label in y_test_str])

    # Create plot with Matplotlib
    fig, ax = plt.subplots(figsize=(10, 8))
    sc = ax.scatter(umap_output[:, 0], umap_output[:, 1], c=colors, s=5)
    ax.set_title("UMAP Visualization", fontsize=14, fontweight="bold")
    ax.set_xlabel("X Dimension", fontsize=12)
    ax.set_ylabel("Y Dimension", fontsize=12)
    ax.grid(True, linestyle='--', alpha=0.5)

    # Add a colorbar to the plot
    sm = plt.cm.ScalarMappable(cmap=colormap, norm=plt.Normalize(vmin=0, vmax=len(unique_labels)-1))
    sm.set_array([])
    cbar = plt.colorbar(sm, ticks=range(len(unique_labels)), ax=ax)
    cbar.ax.set_yticklabels(unique_labels)
    cbar.set_label("MMSIs")
    plt.show()
visualize_intermediate_representations(best_model, x_test[:10000], mmsi_test[:10000], n_neighbors=10, min_dist=0.5)
```

## Final Considerations

GRUs can effectively forecast vessel trajectories but have notable downsides. A primary limitation is their struggle with long-term dependencies due to the vanishing gradient problem, causing the loss of relevant information from earlier time steps. This makes capturing long-term patterns in vessel trajectories challenging. Additionally, GRUs are computationally expensive with large datasets and long sequences, resulting in longer training times and higher memory use. While outperforming basic RNNs, they may not always surpass advanced architectures like LSTMs or Transformer models. Furthermore, the interpretability of GRU-based models is a challenge, which can hinder their adoption in safety-critical applications like vessel trajectory forecasting.
