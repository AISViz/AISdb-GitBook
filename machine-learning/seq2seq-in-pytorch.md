---
icon: arrow-right-arrow-left
---

# seq2seq in PyTorch

**Sequence to Sequence using Torch**

Vessel trajectories are a type of geospatial temporal data derived from AIS (Automatic Identification System) signals. In this tutorial, we will go over the most common Machine Learning Library to process and model AIS trajectory data.

We will begin with **PyTorch**, a widely used deep learning library designed for building and training neural networks. Specifically, we will implement a recurrent neural network using **LSTM (Long Short-Term Memory)** to model sequential patterns in vessel movements.

We will utilize **AISdb**, a dedicated framework for querying, filtering, and preprocessing vessel trajectory data, to streamline data preparation for machine learning workflows.

## Setting Up Our Tools

First, let's import the libraries we'll be using throughout this tutorial. Our main tools will be **NumPy** and **PyTorch**, along with a few other standard libraries for data handling, model building, and visualization.

* `pandas`, `numpy`: for handling tables and arrays
* `torch`: for building and training deep learning models
* `sklearn`: for data splitting and evaluation utilities
* `matplotlib`: for visualizing model performance and outputs

```python
import io
import json
import random
from collections import defaultdict
from datetime import datetime, timedelta

import matplotlib.pyplot as plt
import numpy as np
import pyproj
import torch
import torch.nn as nn
import torch.nn.functional as F
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler, MinMaxScaler, RobustScaler
from torch import nn
from torch.utils.data import DataLoader, TensorDataset

import joblib
import aisdb
from aisdb import DBConn, database
from aisdb.database import sqlfcn_callbacks
from aisdb.database.sqlfcn_callbacks import in_timerange
from aisdb.database.dbconn import PostgresDBConn
from aisdb.track_gen import TrackGen

import cartopy.crs as ccrs
import cartopy.feature as cfeature
```

Assuming you have the database ready, you can replace the file path and establish a connection.&#x20;

We have processed a [sample SQLite database](https://github.com/AISViz/Tutorials/blob/main/example_data.db) containing open-source AIS data from Marine Cadastre, covering January to March near Maine, United States.

To generate the query using AISdb, we use the [DBQuery](https://aisviz.gitbook.io/documentation/tutorials/data-querying) function. All you have to change here is the DB\_CONNECTION , START\_DATE, END\_DATE and the bounding coordinates.

Sample coordinates look like this on the map:

<figure><img src="../.gitbook/assets/image (45).png" alt=""><figcaption></figcaption></figure>

```python
DB_CONNECTION = "/home/sqlite_database_file.db" # replace with your data path
START_DATE = datetime(2018, 8, 1, hour=0) # starting at 12 midnight on 1st August 2018
END_DATE = datetime(2018, 8, 4, hour=2) #Ending at 2:00 am on 3rd August 2018
XMIN, YMIN, XMAX, YMAX =-64.828126,46.113933,-58.500001,49.619290 #Sample coordinates - x refers to longitude and y to latitude

# database connection
dbconn = DBConn(dbpath=DB_CONNECTION)

# Generating query to extract data between a given time range
qry = aisdb.DBQuery(dbconn=dbconn, callback=in_timerange,
                      start = START_DATE,
                      end=END_DATE,
                      xmin=XMIN, xmax=XMAX, ymin=YMIN, ymax=YMAX,
                      )
rowgen = qry.gen_qry(verbose=True) # generating query
tracks = TrackGen(rowgen, decimate=False) # Convert rows into tracks
# rowgen_ = qry.gen_qry(reaggregate_static=True, verbose=True) if you want metadata

#To not get an overfitted model, lets chose data from a completely different date to test

TEST_START_DATE =  datetime(2018, 8, 5, hour=0)
TEST_END_DATE =  datetime(2018,8,6, hour= 8)

test_qry = aisdb.DBQuery(dbconn=dbconn, callback=in_timerange,
                             start = TEST_START_DATE,
                             end = TEST_END_DATE,
                             xmin=XMIN, xmax=XMAX, ymin=YMIN, ymax=YMAX
                        )
test_tracks = TrackGen( (test_qry.gen_qry(verbose=True)), decimate=False)
```

We use `pyproj` for the metric projection of the latitude and longitude values. Learn more [here](https://pyproj4.github.io/pyproj/stable/).

```python
# --- Projection: Lat/Lon -> Cartesian (meters) ---
proj = pyproj.Proj(proj='utm', zone=20, ellps='WGS84')
```

## Preprocessing

We follow the listed steps to prepross the queried trajectory data:

* Remove pings wrt to speed
* encoding of tracks given a threshold
* interpolation according to time (5 mins here)
* group data based on mmsi
* filter out mmsi's with less than 100 points
* Convert lat lon to x & y on cartesian plane using pyroj
* Use the sin cos value of cog as its a 360 degree value
* drop NaN values
* apply scaling to ensure value are normalized

The steps above are wrapped into the function defined as:

```python
def preprocess_aisdb_tracks(tracks_gen, proj, sog_scaler=None, feature_scaler=None, fit_scaler=False):
    # Keep as generator for AISdb functions
    tracks_gen = aisdb.remove_pings_wrt_speed(tracks_gen, 0.1)
    tracks_gen = aisdb.encode_greatcircledistance(tracks_gen,
                                                  distance_threshold=50000,
                                                  minscore=1e-5,
                                                  speed_threshold=50)
    tracks_gen = aisdb.interp_time(tracks_gen, step=timedelta(minutes=5))

    # Convert generator to list AFTER all AISdb steps
    tracks = list(tracks_gen)

    # Group by MMSI
    tracks_by_mmsi = defaultdict(list)
    for track in tracks:
        tracks_by_mmsi[track['mmsi']].append(track)

    # Keep only MMSIs with tracks >= 100 points
    valid_tracks = []
    for mmsi, mmsi_tracks in tracks_by_mmsi.items():
        if all(len(t['time']) >= 100 for t in mmsi_tracks):
            valid_tracks.extend(mmsi_tracks)

    # Convert to DataFrame
    rows = []
    for track in valid_tracks:
        mmsi = track['mmsi']
        sog = track.get('sog', [np.nan]*len(track['time']))
        cog = track.get('cog', [np.nan]*len(track['time']))

        for i in range(len(track['time'])):
            x, y = proj(track['lon'][i], track['lat'][i])
            cog_rad = np.radians(cog[i]) if cog[i] is not None else np.nan
            rows.append({
                'mmsi': mmsi,
                'x': x,
                'y': y,
                'sog': sog[i],
                'cog_sin': np.sin(cog_rad) if not np.isnan(cog_rad) else np.nan,
                'cog_cos': np.cos(cog_rad) if not np.isnan(cog_rad) else np.nan,
                'timestamp': pd.to_datetime(track['time'][i], errors='coerce')
            })

    df = pd.DataFrame(rows)

    # Drop rows with NaNs
    df = df.replace([np.inf, -np.inf], np.nan)
    df = df.dropna(subset=['x', 'y', 'sog', 'cog_sin', 'cog_cos'])

    # Scale features
    feature_cols = ['x', 'y', 'sog','cog_sin','cog_cos']  # only scale directions
    if fit_scaler:
        sog_scaler = RobustScaler()
        df['sog_scaled'] = sog_scaler.fit_transform(df[['sog']])
        feature_scaler = RobustScaler()
        df[feature_cols] = feature_scaler.fit_transform(df[feature_cols])
    else:
        df['sog_scaled'] = sog_scaler.transform(df[['sog']])
        df[feature_cols] = feature_scaler.transform(df[feature_cols])

    return df, sog_scaler, feature_scaler
```

Next, we process all vessel tracks and split them into training and test sets, which are used for model training and evaluation.

```python
train_qry = aisdb.DBQuery(dbconn=dbconn, callback=in_timerange,
                          start=START_DATE, end=END_DATE,
                          xmin=XMIN, xmax=XMAX, ymin=YMIN, ymax=YMAX)
train_gen = TrackGen(train_qry.gen_qry(verbose=True), decimate=False)

test_qry = aisdb.DBQuery(dbconn=dbconn, callback=in_timerange,
                         start=TEST_START_DATE, end=TEST_END_DATE,
                         xmin=XMIN, xmax=XMAX, ymin=YMIN, ymax=YMAX)
test_gen = TrackGen(test_qry.gen_qry(verbose=True), decimate=False)

# --- Preprocess ---
train_df, sog_scaler, feature_scaler = preprocess_aisdb_tracks(train_gen, proj, fit_scaler=True)
test_df, _, _ = preprocess_aisdb_tracks(test_gen, proj, sog_scaler=sog_scaler,
                                        feature_scaler=feature_scaler, fit_scaler=False)
```

## Create Sequences

For geospatial-temporal data, we typically use a sliding window approach, where each trajectory is segmented into input sequences of length _X_ to predict the next _Y_ steps. In this tutorial, we set _X = 80_ and _Y = 2_.

```python
def create_sequences(df, features, input_size=80, output_size=2, step=1):
    X_list, Y_list = [], []
    for mmsi in df['mmsi'].unique():
        sub = df[df['mmsi']==mmsi].sort_values('timestamp')[features].to_numpy()
        for i in range(0, len(sub)-input_size-output_size+1, step):
            X_list.append(sub[i:i+input_size])
            Y_list.append(sub[i+input_size:i+input_size+output_size])
    return torch.tensor(X_list, dtype=torch.float32), torch.tensor(Y_list, dtype=torch.float32)

features = ['x','y','cog_sin','cog_cos','sog_scaled']

mmsis = train_df['mmsi'].unique()
train_mmsi, val_mmsi = train_test_split(mmsis, test_size=0.2, random_state=42, shuffle=True)

train_X, train_Y = create_sequences(train_df[train_df['mmsi'].isin(train_mmsi)], features)
val_X, val_Y     = create_sequences(train_df[train_df['mmsi'].isin(val_mmsi)], features)
test_X, test_Y   = create_sequences(test_df, features)
```

We then save all this data as well as the scalers (we'll use this towards the end in evaluation)

```python
# --- save datasets ---
torch.save({
    'train_X': train_X, 'train_Y': train_Y,
    'val_X': val_X, 'val_Y': val_Y,
    'test_X': test_X, 'test_Y': test_Y
}, 'datasets_seq2seq_cartesian5.pt')

# --- save scalers ---
joblib.dump(feature_scaler, "feature_scaler.pkl")
joblib.dump(sog_scaler, "sog_scaler.pkl")

# --- save projection parameters ---
proj_params = {'proj': 'utm', 'zone': 20, 'ellps': 'WGS84'}
with open("proj_params.json", "w") as f:
    json.dump(proj_params, f)

```

## Load Data

Now we can load the data and start experimenting with it. The same data can also be reused across different models we want to explore.

```python
data = torch.load('datasets_seq2seq_cartesian5.pt')

# scalers
feature_scaler = joblib.load("feature_scaler.pkl")
sog_scaler = joblib.load("sog_scaler.pkl")

# projection
with open("proj_params.json", "r") as f:
    proj_params = json.load(f)
proj = pyproj.Proj(**proj_params)

train_ds = TensorDataset(data['train_X'], data['train_Y'])
val_ds = TensorDataset(data['val_X'], data['val_Y'])
test_ds = TensorDataset(data['test_X'], data['test_Y'])

batch_size = 64
train_dl = DataLoader(train_ds, batch_size=batch_size, shuffle=True)
val_dl = DataLoader(val_ds, batch_size=batch_size, shuffle=False)
test_dl = DataLoader(test_ds, batch_size=batch_size)
```

## Machine Learning Model - Long Short Term Memory (LSTM)

We use an attention-based encoder–decoder LSTM model for trajectory prediction. The model has two layers and incorporates **teacher forcing**, a strategy where the decoder is occasionally fed the ground-truth values during training. This helps stabilize learning and prevents the model from drifting too far when making multi-step predictions.&#x20;

```python
class Seq2SeqLSTM(nn.Module):
    def __init__(self, input_size, hidden_size, input_steps, output_steps):
        super().__init__()
        self.input_steps = input_steps
        self.output_steps = output_steps

        # Encoder
        self.encoder = nn.LSTM(input_size, hidden_size, num_layers=2, dropout=0.3, batch_first=True)

        # Decoder
        self.decoder = nn.LSTMCell(input_size, hidden_size)
        self.attn = nn.Linear(hidden_size + hidden_size, input_steps)  # basic Bahdanau-ish
        self.attn_combine = nn.Linear(hidden_size + input_size, input_size)

        # Output projection (predict residual proposal for all features)
        self.output_layer = nn.Sequential(
            nn.Linear(hidden_size, hidden_size // 2),
            nn.ReLU(),
            nn.Linear(hidden_size // 2, input_size)
        )

    def forward(self, x, target_seq=None, teacher_forcing_ratio=0.5):
        """
        x: [B, T_in, F]
        target_seq: [B, T_out, F] (optional, for teacher forcing)
        returns: [B, T_out, F] predicted absolute features (with x,y = last_obs + residuals)
        """
        batch_size = x.size(0)
        encoder_outputs, (h, c) = self.encoder(x)          # encoder_outputs: [B, T_in, H]
        h, c = h[-1], c[-1]                                # take final layer states -> [B, H]

        last_obs = x[:, -1, :]     # [B, F] last observed features (scaled)
        last_xy  = last_obs[:, :2] # [B, 2]

        decoder_input = last_obs   # start input is last observed feature vector
        outputs = []

        for t in range(self.output_steps):
            # attention weights over encoder outputs: combine h and c to get context
            attn_weights = torch.softmax(self.attn(torch.cat((h, c), dim=1)), dim=1)  # [B, T_in]
            context = torch.bmm(attn_weights.unsqueeze(1), encoder_outputs).squeeze(1)  # [B, H]

            dec_in = torch.cat((decoder_input, context), dim=1)  # [B, F+H]
            dec_in = self.attn_combine(dec_in)                   # [B, F]

            h, c = self.decoder(dec_in, (h, c))                  # h: [B, H]

            residual = self.output_layer(h)                      # [B, F] -- residual proposal for all features

            # Apply residual only to x,y (first two dims); other features are predicted directly
            out_xy = residual[:, :2] + last_xy                   # absolute x,y = last_xy + residual
            out_rest = residual[:, 2:]                           # predicted auxiliary features
            out = torch.cat([out_xy, out_rest], dim=1)           # [B, F]

            outputs.append(out.unsqueeze(1))                     # accumulate

            # Teacher forcing or autoregressive input
            if self.training and (target_seq is not None) and (t < target_seq.size(1)) and (random.random() < teacher_forcing_ratio):
                decoder_input = target_seq[:, t, :]  # teacher forcing uses ground truth step (scaled)
                # update last_xy to ground truth (so if teacher forced, we align residual base)
                last_xy = decoder_input[:, :2]
            else:
                decoder_input = out
                last_xy = out[:, :2]                  # update last_xy to predicted absolute xy

        return torch.cat(outputs, dim=1)  # [B, T_out, F]

```

## Auxiliary Loss Components

Two auxiliary functions are introduced to augment the original MSE loss. These additional terms are designed to better preserve the physical consistency and structural shape of the predicted trajectory.

```python
def weighted_coord_loss(pred, target, coord_weight=5.0, reduction='mean'):
    """
    Give higher weight to coordinate errors (first two dims).
    pred/target: [B, T, F]
    """
    coord_loss = F.smooth_l1_loss(pred[..., :2], target[..., :2], reduction=reduction)
    aux_loss   = F.smooth_l1_loss(pred[..., 2:], target[..., 2:], reduction=reduction)
    return coord_weight * coord_loss + aux_loss


def xy_smoothness_loss(seq_full):
    """
    Smoothness penalty only on x,y channels of a full sequence.
    seq_full: [B, T, F] (F >= 2)
    Returns scalar L2 of second differences on x,y.
    """
    xy = seq_full[..., :2]              # [B, T, 2]
    v = xy[:, 1:, :] - xy[:, :-1, :]    # velocity [B, T-1, 2]
    a = v[:, 1:, :] - v[:, :-1, :]      # acceleration [B, T-2, 2]
    return (v**2).mean() * 0.05 + (a**2).mean() * 0.5

```

## Model Training

Once the model is defined, the next step is to train it on our prepared dataset. Training involves iteratively feeding input sequences to the model, comparing its predictions against the ground truth, and updating the weights to reduce the error.

In our case, the loss function combines:

* a **data term** (based on weighted coordinate errors and auxiliary features), and
* a **smoothness penalty** (to encourage realistic vessel movement and reduce jitter in the predicted trajectory).

```python
def train_model(model, loader, val_dl, optimizer, device, epochs=50,
                coord_weight=5.0, smooth_w_init=1e-3):
    best_loss = float('inf')
    best_state = None

    for epoch in range(epochs):
        model.train()
        total_loss = total_data_loss = total_smooth_loss = 0.0

        for batch_x, batch_y in loader:
            batch_x = batch_x.to(device)   # [B, T_in, F]
            batch_y = batch_y.to(device)   # [B, T_out, F]

            # --- Convert targets to residuals ---
            # residual = y[t] - y[t-1], relative to last obs (x[:,-1])
            y_start = batch_x[:, -1:, :2]   # last observed xy [B,1,2]
            residual_y = batch_y.clone()
            residual_y[..., :2] = batch_y[..., :2] - torch.cat([y_start, batch_y[:, :-1, :2]], dim=1)

            optimizer.zero_grad()

            # Model predicts residuals
            pred_residuals = model(batch_x, target_seq=residual_y, teacher_forcing_ratio=0.5)  # [B, T_out, F]

            # Data loss on residuals (only xy are residualized, other features can stay as-is)
            loss_data = weighted_coord_loss(pred_residuals, residual_y, coord_weight=coord_weight)

            # Smoothness: reconstruct absolute xy sequence first
            pred_xy = torch.cumsum(pred_residuals[..., :2], dim=1) + y_start
            full_seq = torch.cat([batch_x[..., :2], pred_xy], dim=1)   # [B, T_in+T_out, 2]
            loss_smooth = xy_smoothness_loss(full_seq)

            # Decay smooth weight slightly over epochs
            smooth_weight = smooth_w_init * max(0.1, 1.0 - epoch / 30.0)
            loss = loss_data + smooth_weight * loss_smooth

            # Stability guard
            if torch.isnan(loss) or torch.isinf(loss):
                print("Skipping batch due to NaN/inf loss.")
                continue

            loss.backward()
            optimizer.step()

            total_loss += loss.item()
            total_data_loss += loss_data.item()
            total_smooth_loss += loss_smooth.item()

        avg_loss = total_loss / len(loader)
        print(f"Epoch {epoch+1:02d} | Total: {avg_loss:.6f} | Data: {total_data_loss/len(loader):.6f} | Smooth: {total_smooth_loss/len(loader):.6f}")

        # Validation
        model.eval()
        val_loss = 0.0
        with torch.no_grad():
            for xb, yb in val_dl:
                xb, yb = xb.to(device), yb.to(device)

                # Compute residuals for validation
                y_start = xb[:, -1:, :2]
                residual_y = yb.clone()
                residual_y[..., :2] = yb[..., :2] - torch.cat([y_start, yb[:, :-1, :2]], dim=1)

                pred_residuals = model(xb, target_seq=residual_y, teacher_forcing_ratio=0.0)

                data_loss = weighted_coord_loss(pred_residuals, residual_y, coord_weight=coord_weight)

                pred_xy = torch.cumsum(pred_residuals[..., :2], dim=1) + y_start
                full_seq = torch.cat([xb[..., :2], pred_xy], dim=1)
                loss_smooth = xy_smoothness_loss(full_seq)

                val_loss += (data_loss + smooth_weight * loss_smooth).item()

        val_loss /= len(val_dl)
        print(f"           Val Loss: {val_loss:.6f}")

        if val_loss < best_loss:
            best_loss = val_loss
            best_state = model.state_dict()

    if best_state is not None:
        torch.save(best_state, "best_model_seq2seq_residual_xy_08302.pth")
        print("✅ Best model saved")

```

```python
# Setting Seeds are extrememly important in research purposes to make sure the results are reproducible
SEED = 42
torch.manual_seed(SEED)
np.random.seed(SEED)
random.seed(SEED)
torch.cuda.manual_seed_all(SEED)
torch.backends.cudnn.deterministic = True
torch.backends.cudnn.benchmark = False

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
train_X = data['train_X']
input_steps = 80
output_steps = 2
# Collapse batch and time dims to compute global min for each feature
flat_train_X = train_X.view(-1, train_X.shape[-1])  # shape: [N*T, 4]


input_size = 5
hidden_size = 64

model = Seq2SeqLSTM(input_size, hidden_size, input_steps, output_steps).to(device)
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
criterion = nn.HuberLoss()

train_model(model, train_dl,val_dl, optimizer, device)

```

## Model Evaluation

Finally, now that our model has been trained we use an evaluation function to check it in the different dataset we had stores earlier, as well as plot it on a map to see how the trajectory predictions look.\
\
Note- we dont just rely on the accuracy or training/testing results in numbers. There might be chances when the loss is showing in decimals but the coordinates are way far off. That is why we chose to plot it out on a map as well to check the predictions.

```python
def evaluate_with_errors(
    model,
    test_dl,
    proj,
    feature_scaler,
    device,
    num_batches=1,
    outputs_are_residual_xy: bool = True,
    dup_tol: float = 1e-4,
    residual_decode_mode: str = "cumsum",   # "cumsum", "independent", "stdonly"
    residual_std: np.ndarray = None,        # needed if residual_decode_mode="stdonly"
):
    """
    Evaluates model and prints/plots errors.

    Parameters
    ----------
    outputs_are_residual_xy : bool
        If True, model outputs residuals (scaled). Else, absolute coords (scaled).
    residual_decode_mode : str
        - "cumsum": cumulative sum of residuals (default).
        - "independent": treat each residual as absolute offset from last obs.
        - "stdonly": multiply by provided residual_std instead of scaler.scale_.
    residual_std : np.ndarray
        Std dev for decoding (meters), used only if residual_decode_mode="stdonly".
    """
    model.eval()
    errors_all = []

    def inverse_xy_only(xy_scaled, scaler):
        """Inverse transform x,y only (works with StandardScaler/RobustScaler)."""
        if hasattr(scaler, "mean_"):
            center = scaler.mean_[:2]
            scale = scaler.scale_[:2]
        elif hasattr(scaler, "center_"):
            center = scaler.center_[:2]
            scale = scaler.scale_[:2]
        else:
            raise ValueError("Scaler type not supported for XY inversion")
        return xy_scaled * scale + center

    def haversine(lon1, lat1, lon2, lat2):
        """Distance (m) between two lon/lat points using haversine formula."""
        R = 6371000.0
        lon1, lat1, lon2, lat2 = map(np.radians, [lon1, lat1, lon2, lat2])
        dlon = lon2 - lon1
        dlat = lat2 - lat1
        a = np.sin(dlat/2.0)**2 + np.cos(lat1)*np.cos(lat2)*np.sin(dlon/2.0)**2
        return 2 * R * np.arcsin(np.sqrt(a))

    with torch.no_grad():
        batches = 0
        for xb, yb in test_dl:
            xb = xb.to(device); yb = yb.to(device)
            pred = model(xb, teacher_forcing_ratio=0.0)  # [B, T_out, F]

            # first sample for visualization/diagnostics
            input_seq = xb[0].cpu().numpy()
            real_seq  = yb[0].cpu().numpy()
            pred_seq  = pred[0].cpu().numpy()

            input_xy_s = input_seq[:, :2]
            real_xy_s  = real_seq[:, :2]
            pred_xy_s  = pred_seq[:, :2]

            # reconstruct predicted absolute trajectory
            if outputs_are_residual_xy:
                last_obs_xy_s = input_xy_s[-1]
                last_obs_xy_m = inverse_xy_only(last_obs_xy_s, feature_scaler)

                # choose decoding mode
                if residual_decode_mode == "stdonly":
                    if residual_std is None:
                        raise ValueError("residual_std must be provided for 'stdonly' mode")
                    pred_resid_m = pred_xy_s * residual_std
                    pred_xy_m = np.cumsum(pred_resid_m, axis=0) + last_obs_xy_m

                elif residual_decode_mode == "independent":
                    if hasattr(feature_scaler, "scale_"):
                        scale = feature_scaler.scale_[:2]
                    else:
                        raise ValueError("Scaler missing scale_ for residual decoding")
                    pred_resid_m = pred_xy_s * scale
                    pred_xy_m = pred_resid_m + last_obs_xy_m  # no cumsum

                else:  # default: "cumsum"
                    if hasattr(feature_scaler, "scale_"):
                        scale = feature_scaler.scale_[:2]
                    else:
                        raise ValueError("Scaler missing scale_ for residual decoding")
                    pred_resid_m = pred_xy_s * scale
                    pred_xy_m = np.cumsum(pred_resid_m, axis=0) + last_obs_xy_m

            else:
                pred_xy_m = inverse_xy_only(pred_xy_s, feature_scaler)

            # Handle duplicate first target
            trimmed_real_xy_s = real_xy_s.copy()
            dropped_duplicate = False
            if trimmed_real_xy_s.shape[0] >= 1:
                if np.allclose(trimmed_real_xy_s[0], input_xy_s[-1], atol=dup_tol):
                    trimmed_real_xy_s = trimmed_real_xy_s[1:]
                    dropped_duplicate = True

            # Align lengths
            len_pred = pred_xy_m.shape[0]
            len_real = trimmed_real_xy_s.shape[0]
            min_len = min(len_pred, len_real)
            if min_len == 0:
                print("No overlapping horizon — skipping batch.")
                batches += 1
                if batches >= num_batches:
                    break
                else:
                    continue

            pred_xy_m = pred_xy_m[:min_len]
            real_xy_m = inverse_xy_only(trimmed_real_xy_s[:min_len], feature_scaler)
            input_xy_m = inverse_xy_only(input_xy_s, feature_scaler)

            # debug
            print("\nDEBUG (scaled -> unscaled):")
            print("last_obs_scaled:", input_xy_s[-1])
            print("real_first_scaled:", real_xy_s[0])
            print("pred_first_scaled_delta:", pred_xy_s[0])
            print("dropped_duplicate_target:", dropped_duplicate)
            print("pred length:", pred_xy_m.shape[0], "real length:", real_xy_m.shape[0])
            print("last_obs_unscaled:", input_xy_m[-1])
            print("pred_first_unscaled:", pred_xy_m[0])

            # lon/lat conversion
            lon_in,  lat_in   = proj(input_xy_m[:,0],  input_xy_m[:,1],  inverse=True)
            lon_real, lat_real= proj(real_xy_m[:,0],   real_xy_m[:,1],   inverse=True)
            lon_pred, lat_pred= proj(pred_xy_m[:,0],   pred_xy_m[:,1],   inverse=True)

            # comparison table
            print("\n=== Predicted vs True (lat/lon) ===")
            print(f"{'t':>3} | {'lon_true':>9} | {'lat_true':>9} | {'lon_pred':>9} | {'lat_pred':>9} | {'err_m':>9}")
            errors = []
            for t in range(len(lon_real)):
                err_m = haversine(lon_real[t], lat_real[t], lon_pred[t], lat_pred[t])
                errors.append(err_m)
                print(f"{t:3d} | {lon_real[t]:9.5f} | {lat_real[t]:9.5f} | {lon_pred[t]:9.5f} | {lat_pred[t]:9.5f} | {err_m:9.2f}")

            errors_all.append(errors)

            # plot
            fig = plt.figure(figsize=(8, 6))
            ax = plt.axes(projection=ccrs.PlateCarree())
            
            # set extent dynamically around trajectory
            all_lons = np.concatenate([lon_in, lon_real, lon_pred])
            all_lats = np.concatenate([lat_in, lat_real, lat_pred])
            lon_min, lon_max = all_lons.min() - 0.01, all_lons.max() + 0.01
            lat_min, lat_max = all_lats.min() - 0.01, all_lats.max() + 0.01
            ax.set_extent([lon_min, lon_max, lat_min, lat_max], crs=ccrs.PlateCarree())
            
            # add map features
            ax.add_feature(cfeature.COASTLINE)
            ax.add_feature(cfeature.LAND, facecolor="lightgray")
            ax.add_feature(cfeature.OCEAN, facecolor="lightblue")
            
            # plot trajectories
            ax.plot(lon_in,   lat_in,   "o-", label="history", transform=ccrs.PlateCarree(),
                    markersize=6, linewidth=2)
            ax.plot(lon_real, lat_real, "o-", label="true",    transform=ccrs.PlateCarree(),
                    markersize=6, linewidth=2)
            ax.plot(lon_pred, lat_pred, "x--", label="pred",   transform=ccrs.PlateCarree(),
                    markersize=8, linewidth=2)
            
            ax.legend()
            plt.show()


            batches += 1
            if batches >= num_batches:
                break

    # summary
    if errors_all:
        errors_all = np.array(errors_all)
        mean_per_t = errors_all.mean(axis=0)
        print("\n=== Summary (meters) ===")
        for t, v in enumerate(mean_per_t):
            print(f"t={t} mean error: {v:.2f} m")
        print(f"mean over horizon: {errors_all.mean():.2f} m, median: {np.median(errors_all):.2f} m")

```

There are some debugging statements as well to see whether the scaling is right or wrong, the distace error etc. In this Model we have a metric distance error of only 800m.

```python
# --- device ---
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
data = torch.load("datasets_seq2seq_cartesian5.pt")

train_X, train_Y = data['train_X'], data['train_Y']
val_X, val_Y     = data['val_X'], data['val_Y']
test_X, test_Y   = data['test_X'], data['test_Y']


# --- load scalers ---
feature_scaler = joblib.load("feature_scaler.pkl")  # fitted on x,y,sog,cog_sin,cog_cos
sog_scaler     = joblib.load("sog_scaler.pkl")      

# --- load projection ---
with open("proj_params.json", "r") as f:
    proj_params = json.load(f)
proj = pyproj.Proj(**proj_params)

# --- rebuild & load model ---
input_size = train_X.shape[2]   # features per timestep
input_steps = train_X.shape[1]  # seq length in
output_steps = train_Y.shape[1] # seq length out

hidden_size = 64    
num_layers = 2      

best_model = Seq2SeqLSTM(
    input_size=5,
    hidden_size=64,
    input_steps=80,   
    output_steps=2,   
).to(device)

best_model.load_state_dict(torch.load(
    "best_model_seq2seq_residual_xy_08302.pth", map_location=device
))
best_model.eval()



import numpy as np

# helper: inverse xy using your loaded feature_scaler (same as in evaluate)
def inverse_xy_only_np(xy_scaled, scaler):
    if hasattr(scaler, "mean_"):
        center = scaler.mean_[:2]
        scale  = scaler.scale_[:2]
    elif hasattr(scaler, "center_"):
        center = scaler.center_[:2]
        scale  = scaler.scale_[:2]
    else:
        raise ValueError("Scaler type not supported for XY inversion")
    return xy_scaled * scale + center


N = train_X.shape[0]
all_resids = []
for i in range(N):
    last_obs_s = train_X[i, -1, :2]               # scaled
    true_s     = train_Y[i, :, :2]                # scaled (T_out,2)

    last_obs_m = inverse_xy_only_np(last_obs_s, feature_scaler)
    true_m     = inverse_xy_only_np(true_s, feature_scaler)

   
    if true_m.shape[0] == 0:
        continue
    resid0 = true_m[0] - last_obs_m
    if true_m.shape[0] > 1:
        rest = np.diff(true_m, axis=0)
        resids = np.vstack([resid0[None, :], rest])   # shape [T_out, 2]
    else:
        resids = resid0[None, :]

    all_resids.append(resids)

# stack to compute std per axis across all time steps and samples
all_resids_flat = np.vstack(all_resids)  # [sum_T, 2]
residual_std = np.std(all_resids_flat, axis=0)  # [std_dx_m, std_dy_m]
print("Computed residual_std (meters):", residual_std)

evaluate_with_errors(
    best_model, test_dl, proj, feature_scaler, device,
    num_batches=1,
    outputs_are_residual_xy=True,
    residual_decode_mode="stdonly",
    residual_std=residual_std
)


```

## Results

Predicted vs True (lat/lon)&#x20;

<table><thead><tr><th width="64.79998779296875">t</th><th>lon_true</th><th>lon_pred</th><th>lat_true</th><th>lat_pred</th><th>Error (in m)</th></tr></thead><tbody><tr><td>0</td><td>-61.69744</td><td> -61.70585</td><td>43.22816</td><td>43.22385</td><td> 833.31 m</td></tr></tbody></table>

Summary (meters)\
t=0 mean error: 833.31 m\
mean over horizon: 833.31 m, median: 833.31 m



<figure><img src="../.gitbook/assets/image (46).png" alt=""><figcaption></figcaption></figure>



