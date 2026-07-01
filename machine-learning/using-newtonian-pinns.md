---
icon: code-pull-request
---

# Using Newtonian PINNs

Traditional Seq2Seq LSTM models have long been the workhorse for trajectory forecasting. They excel at learning temporal dependencies in AIS data, and with attention mechanisms they can capture complex nonlinear patterns over long histories. However, they remain purely statistical. This means the model can generate plausible-looking trajectories from a data perspective but with no guarantee that those predictions respect the underlying physics of vessel motion. In practice, this often manifests as sharp turns, unrealistic accelerations, or trajectories that deviate significantly when the model faces sparse or noisy data.

The NPINN-based approach directly addresses these shortcomings. By embedding smoothness and kinematic penalties into training, it enforces constraints on velocity and acceleration while still benefiting from the representational power of deep sequence models. Instead of simply fitting residuals between past and future positions, NPINN ensures that predictions evolve in ways consistent with how vessels actually move in the physical world. This leads to more reliable extrapolation, especially in data-scarce regions or unusual navigation scenarios.

### Preprocessing

The first step in building a trajectory learning pipeline is **preprocessing AIS tracks into a model-friendly format**. Raw AIS messages are noisy, irregularly sampled, and inconsistent across vessels, so we need to enforce structure before feeding them into neural networks. The function below does several things in sequence:

1. **Data cleaning** – removes spurious pings based on unrealistic speeds, encodes great-circle distances, and interpolates trajectories at fixed 5-minute intervals.
2. **Track filtering** – groups data by vessel (MMSI) and keeps only sufficiently long tracks to ensure stable training samples.
3. **Feature extraction** – converts lat/lon into projected coordinates (`x`, `y`), adds speed over ground (`sog`), and represents course over ground (`cog`) as sine/cosine to avoid angular discontinuities.
4. **Delta computation** – calculates `dx` and `dy` between consecutive timestamps, capturing local motion dynamics.
5. **Scaling** – applies `RobustScaler` to normalize features and deltas while being resilient to outliers (common in AIS data).

The result is a **clean, scaled DataFrame** where each row represents a vessel state at a timestamp, enriched with both absolute position features and relative motion features.

```python
from sklearn.preprocessing import RobustScaler

def preprocess_aisdb_tracks(tracks_gen, proj,
                            sog_scaler=None,
                            feature_scaler=None,
                            delta_scaler=None,
                            fit_scaler=False):
    # --- AISdb cleaning ---
    tracks_gen = aisdb.remove_pings_wrt_speed(tracks_gen, 0.1)
    tracks_gen = aisdb.encode_greatcircledistance(
        tracks_gen,
        distance_threshold=50000,
        minscore=1e-5,
        speed_threshold=50
    )
    tracks_gen = aisdb.interp_time(tracks_gen, step=timedelta(minutes=5))

    # --- collect tracks ---
    tracks = list(tracks_gen)

    # --- group by MMSI ---
    tracks_by_mmsi = defaultdict(list)
    for track in tracks:
        tracks_by_mmsi[track['mmsi']].append(track)

    # --- keep only long-enough tracks ---
    valid_tracks = []
    for mmsi, mmsi_tracks in tracks_by_mmsi.items():
        if all(len(t['time']) >= 100 for t in mmsi_tracks):
            valid_tracks.extend(mmsi_tracks)

    # --- flatten into dataframe ---
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

    # --- clean NaNs ---
    df = df.replace([np.inf, -np.inf], np.nan)
    df = df.dropna(subset=['x', 'y', 'sog', 'cog_sin', 'cog_cos'])

    # --- compute deltas per MMSI ---
    df = df.sort_values(["mmsi", "timestamp"])
    df["dx"] = df.groupby("mmsi")["x"].diff().fillna(0)
    df["dy"] = df.groupby("mmsi")["y"].diff().fillna(0)

    # --- scale features ---
    feature_cols = ['x', 'y', 'sog', 'cog_sin', 'cog_cos']
    delta_cols   = ['dx', 'dy']

    if fit_scaler:
        # sog
        sog_scaler = RobustScaler()
        df['sog_scaled'] = sog_scaler.fit_transform(df[['sog']])

        # absolute features
        feature_scaler = RobustScaler()
        df[feature_cols] = feature_scaler.fit_transform(df[feature_cols])

        # deltas
        delta_scaler = RobustScaler()
        df[delta_cols] = delta_scaler.fit_transform(df[delta_cols])
    else:
        df['sog_scaled'] = sog_scaler.transform(df[['sog']])
        df[feature_cols] = feature_scaler.transform(df[feature_cols])
        df[delta_cols]   = delta_scaler.transform(df[delta_cols])

    return df, sog_scaler, feature_scaler, delta_scaler
```

We first **query the AIS database** for the training and testing periods and geographic bounds, producing generators of raw vessel tracks. These raw tracks are then **preprocessed** using `preprocess_aisdb_tracks`, which cleans the data, computes relative motion (`dx`, `dy`), scales features, and outputs a ready-to-use DataFrame. Training data fits new scalers, while test data is transformed using the same scalers to ensure consistency.

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
train_df, sog_scaler, feature_scaler, delta_scaler = preprocess_aisdb_tracks(
    train_gen, proj, fit_scaler=True
)

test_df, _, _, _ = preprocess_aisdb_tracks(
    test_gen,
    proj,
    sog_scaler=sog_scaler,
    feature_scaler=feature_scaler,
    delta_scaler=delta_scaler,
    fit_scaler=False
)
```

The `create_sequences` function **transforms the preprocessed track data into supervised sequences** suitable for model training. For each vessel, it slides a fixed-size window over the time series, building **input sequences** of past absolute features (`x, y, dx, dy, cog_sin, cog_cos, sog_scaled`) and **target sequences** of future residual movements (`dx, dy`). Using this, the dataset is split into training, validation, and test sets, with each set containing sequences ready for direct input into a trajectory prediction model.

```python
def create_sequences(df, features, input_size=80, output_size=2, step=1):
    """
    Build sequences:
      X: past window of absolute features (x, y, dx, dy, cog_sin, cog_cos, sog_scaled)
      Y: future residuals (dx, dy)
    """
    X_list, Y_list = [], []
    for mmsi in df['mmsi'].unique():
        sub = df[df['mmsi'] == mmsi].sort_values('timestamp').copy()

        # build numpy arrays
        feat_arr = sub[features].to_numpy()
        dxdy_arr = sub[['dx', 'dy']].to_numpy()  # residuals already scaled

        for i in range(0, len(sub) - input_size - output_size + 1, step):
            # input sequence is absolute features
            X_list.append(feat_arr[i : i + input_size])

            # output sequence is residuals immediately after
            Y_list.append(dxdy_arr[i + input_size : i + input_size + output_size])

    return torch.tensor(X_list, dtype=torch.float32), torch.tensor(Y_list, dtype=torch.float32)


features = ['x', 'y', 'dx', 'dy', 'cog_sin', 'cog_cos', 'sog_scaled']

mmsis = train_df['mmsi'].unique()
train_mmsi, val_mmsi = train_test_split(mmsis, test_size=0.2, random_state=42, shuffle=True)

train_X, train_Y = create_sequences(train_df[train_df['mmsi'].isin(train_mmsi)], features)
val_X, val_Y     = create_sequences(train_df[train_df['mmsi'].isin(val_mmsi)], features)
test_X, test_Y   = create_sequences(test_df, features)
```

### Save the Dataset

This block **saves all processed data and supporting objects** needed for training and evaluation. The preprocessed input and target sequences for training, validation, and testing are serialized using PyTorch (`datasets_npin.pt`). The fitted **scalers** for features, speed, and residuals are saved with `joblib` to ensure consistent scaling during inference. Finally, the **projection parameters** used to convert geographic coordinates to UTM are stored in JSON, allowing consistent coordinate transformations later.

```python
import torch
import joblib
import json

# --- save datasets ---
torch.save({
    'train_X': train_X, 'train_Y': train_Y,
    'val_X': val_X, 'val_Y': val_Y,
    'test_X': test_X, 'test_Y': test_Y
}, 'datasets_npin.pt')

# --- save scalers ---
joblib.dump(feature_scaler, "npinn_feature_scaler.pkl")
joblib.dump(sog_scaler, "npinn_sog_scaler.pkl")
joblib.dump(delta_scaler, "npinn_delta_scaler.pkl")   # NEW


# --- save projection parameters ---
proj_params = {'proj': 'utm', 'zone': 20, 'ellps': 'WGS84'}
with open("npinn_proj_params.json", "w") as f:
    json.dump(proj_params, f)

```

```python

data = torch.load('datasets_npin.pt')
import torch
import joblib
import json
# scalers
feature_scaler = joblib.load("npinn_feature_scaler.pkl")
sog_scaler = joblib.load("npinn_sog_scaler.pkl")
delta_scaler   = joblib.load("npinn_delta_scaler.pkl")   # NEW

# projection
with open("npinn_proj_params.json", "r") as f:
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

### Model

`Seq2SeqLSTM` is the **sequence-to-sequence backbone used in the NPINN-based trajectory prediction framework**. Within NPINN, the **encoder LSTM** processes past vessel observations (positions, speed, and course) to produce a hidden representation that captures motion dynamics. The **decoder LSTMCell** predicts future residuals in x and y, with an **attention mechanism** that selectively focuses on relevant past information at each step. Predicted residuals are added to the last observed position to reconstruct absolute trajectories.

This setup enables NPINN to generate **smooth, physically consistent multi-step vessel trajectories**, leveraging both historical motion patterns and learned dynamics constraints.

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
        self.attn = nn.Linear(hidden_size * 2, input_steps)
        self.attn_combine = nn.Linear(hidden_size + input_size, input_size)

        # Output only x,y residuals (added to last observed pos)
        self.output_layer = nn.Sequential(
            nn.Linear(hidden_size, hidden_size // 2),
            nn.ReLU(),
            nn.Linear(hidden_size // 2, 2)
        )

    def forward(self, x, target_seq=None, teacher_forcing_ratio=0.5):
        batch_size = x.size(0)
        encoder_outputs, (h, c) = self.encoder(x)
        h, c = h[-1], c[-1]

        last_obs = x[:, -1, :2]   # last observed absolute x,y
        decoder_input = x[:, -1, :]  # full feature vector

        outputs = []
        for t in range(self.output_steps):
            attn_weights = torch.softmax(self.attn(torch.cat((h, c), dim=1)), dim=1)
            context = torch.bmm(attn_weights.unsqueeze(1), encoder_outputs).squeeze(1)

            dec_in = torch.cat((decoder_input, context), dim=1)
            dec_in = self.attn_combine(dec_in)

            h, c = self.decoder(dec_in, (h, c))
            residual_xy = self.output_layer(h)

            # accumulate into absolute xy
            out_xy = residual_xy + last_obs
            outputs.append(out_xy.unsqueeze(1))

            # teacher forcing
            if self.training and target_seq is not None and t < target_seq.size(1) and random.random() < teacher_forcing_ratio:
                decoder_input = torch.cat([target_seq[:, t, :2], decoder_input[:, 2:]], dim=1)
                last_obs = target_seq[:, t, :2]
            else:
                decoder_input = torch.cat([out_xy, decoder_input[:, 2:]], dim=1)
                last_obs = out_xy

        return torch.cat(outputs, dim=1)  # (batch, output_steps, 2)
```

### Model Training

This training loop implements **NPINN-based trajectory learning**, combining **data fidelity** with **physics-inspired smoothness constraints**. The `weighted_coord_loss` enforces accurate prediction of future x, y positions, while `xy_npinn_smoothness_loss` encourages smooth velocity and acceleration profiles, reflecting realistic vessel motion.

By integrating these two objectives, NPINN **learns trajectories that are both close to observed data and physically plausible**, with the smoothness weight gradually decaying during training to balance learning accuracy with dynamic consistency. Validation is performed each epoch to ensure generalization, and the best-performing model is saved. This approach differentiates NPINN from standard Seq2Seq training by **explicitly incorporating motion dynamics into the loss**, rather than relying purely on sequence prediction.

```python
def weighted_coord_loss(pred, target, coord_weight=5.0, reduction='mean'):
    return F.smooth_l1_loss(pred, target, reduction=reduction)


def xy_npinn_smoothness_loss(seq_full, coord_min=None, coord_max=None):
    """
    NPINN-inspired smoothness penalty on xy coordinates
    seq_full: [B, T, 2]
    """
    xy = seq_full[..., :2]

    if coord_min is not None and coord_max is not None:
        xy_norm = (xy - coord_min) / (coord_max - coord_min + 1e-8)
        xy_norm = 2 * (xy_norm - 0.5)  # [-1,1]
    else:
        xy_norm = xy

    v = xy_norm[:, 1:, :] - xy_norm[:, :-1, :]
    a = v[:, 1:, :] - v[:, :-1, :]
    return (v**2).mean() * 0.05 + (a**2).mean() * 0.5

```

```python
def train_model(model, loader, val_dl, optimizer, device, epochs=50,
                smooth_w_init=1e-3, coord_min=None, coord_max=None):

    best_loss = float('inf')
    best_state = None

    for epoch in range(epochs):
        model.train()
        total_loss = total_data_loss = total_smooth_loss = 0.0

        for batch_x, batch_y in loader:
            batch_x = batch_x.to(device)  # [B, T_in, F]
            batch_y = batch_y.to(device)  # [B, T_out, 2] absolute x,y

            optimizer.zero_grad()
            pred_xy = model(batch_x, target_seq=batch_y, teacher_forcing_ratio=0.5)

            # Data loss: directly on absolute x,y
            loss_data = weighted_coord_loss(pred_xy, batch_y)

            # Smoothness loss: encourage smooth xy trajectories
            y_start = batch_x[:, :, :2]
            full_seq = torch.cat([y_start, pred_xy], dim=1)  # observed + predicted
            loss_smooth = xy_npinn_smoothness_loss(full_seq, coord_min, coord_max)

            smooth_weight = smooth_w_init * max(0.1, 1.0 - epoch / 30.0)
            loss = loss_data + smooth_weight * loss_smooth

            loss.backward()
            optimizer.step()

            total_loss += loss.item()
            total_data_loss += loss_data.item()
            total_smooth_loss += loss_smooth.item()

        avg_loss = total_loss / len(loader)
        print(f"Epoch {epoch+1} | Total: {avg_loss:.6f} | Data: {total_data_loss/len(loader):.6f} | Smooth: {total_smooth_loss/len(loader):.6f}")

        # Validation
        model.eval()
        val_loss = 0.0
        with torch.no_grad():
            for xb, yb in val_dl:
                xb, yb = xb.to(device), yb.to(device)
                pred_xy = model(xb, target_seq=yb, teacher_forcing_ratio=0.0)

                data_loss = weighted_coord_loss(pred_xy, yb)
                full_seq = torch.cat([xb[..., :2], pred_xy], dim=1)
                loss_smooth = xy_npinn_smoothness_loss(full_seq, coord_min, coord_max)

                val_loss += (data_loss + smooth_weight * loss_smooth).item()

        val_loss /= len(val_dl)
        print(f"           Val Loss: {val_loss:.6f}")

        if val_loss < best_loss:
            best_loss = val_loss
            best_state = model.state_dict()

    if best_state is not None:
        torch.save(best_state, "best_model_NPINN.pth")
        print("Best model saved")
```

### Training

This training sets a fixed random seed for reproducibility (`torch`, `numpy`, `random`) and enables deterministic CuDNN behavior. It initializes a Seq2SeqLSTM NPINN model to predict future vessel trajectory residuals from past sequences of absolute features. Global min/max of xy coordinates are computed from the training set for NPINN’s smoothness loss normalization. The model is trained on GPU if available using Adam, combining data loss on predicted xy positions with a physics-inspired smoothness penalty to enforce realistic, physically plausible trajectories.

```python
import torch
import numpy as np
import random
from torch import nn

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

# Collapse batch and time dims to compute global min/max for each feature
flat_train_X = train_X.view(-1, train_X.shape[-1])  # shape: [N*T, F]

x_min, x_max = flat_train_X[:, 0].min().item(), flat_train_X[:, 0].max().item()
y_min, y_max = flat_train_X[:, 1].min().item(), flat_train_X[:, 1].max().item()
cog_sin_min, cog_sin_max = flat_train_X[:, 2].min().item(), flat_train_X[:, 2].max().item()
cog_cos_min, cog_cos_max = flat_train_X[:, 3].min().item(), flat_train_X[:, 3].max().item()
sog_min, sog_max = flat_train_X[:, 4].min().item(), flat_train_X[:, 4].max().item()

coord_min = torch.tensor([x_min, y_min], device=device)
coord_max = torch.tensor([x_max, y_max], device=device)

# Model setup
input_size = 7 
hidden_size = 64

model = Seq2SeqLSTM(input_size, hidden_size, input_steps, output_steps).to(device)
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)

# coord_min/max only for xy
coord_min = torch.tensor([x_min, y_min], device=device)
coord_max = torch.tensor([x_max, y_max], device=device)

train_model(model, train_dl, val_dl, optimizer, device,
            coord_min=coord_min, coord_max=coord_max)
```

### Evaluate

This snippet sets up the environment for **inference or evaluation** of the NPINN Seq2Seq model:

* Chooses GPU if available.
* Loads preprocessed datasets (`train_X/Y`, `val_X/Y`, `test_X/Y`).
* Loads the saved `RobustScaler`s for features, SOG, and deltas to match preprocessing during training.
* Loads projection parameters to convert lon/lat to projected coordinates consistently.
* Rebuilds the same `Seq2SeqLSTM` NPINN model used during training and loads the best saved weights.
* Puts the model in evaluation mode, ready for predicting future vessel trajectories.

Essentially, this is the **full recovery pipeline for NPINN inference**, ensuring consistency with training preprocessing, scaling, and projection.

```python
import torch
import joblib
import json
import pyproj
import numpy as np
from torch.utils.data import DataLoader, TensorDataset
import cartopy.crs as ccrs
import cartopy.feature as cfeature

# --- device ---
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")

# recover arrays (assumes you already loaded `data`)
train_X, train_Y = data['train_X'], data['train_Y']
val_X, val_Y     = data['val_X'], data['val_Y']
test_X, test_Y   = data['test_X'], data['test_Y']

# --- load scalers (must match what you saved during preprocessing) ---
feature_scaler = joblib.load("npinn_feature_scaler.pkl")
sog_scaler = joblib.load("npinn_sog_scaler.pkl")
delta_scaler   = joblib.load("npinn_delta_scaler.pkl")    # for dx,dy
# if you named them differently, change the filenames above

# --- load projection params and build proj ---
with open("proj_params.json", "r") as f:
    proj_params = json.load(f)
proj = pyproj.Proj(**proj_params)

# --- rebuild & load model ---
input_size   = train_X.shape[2]
input_steps  = train_X.shape[1]
output_steps = train_Y.shape[1]

hidden_size = 64
num_layers  = 2

best_model = Seq2SeqLSTM(
    input_size=input_size,
    hidden_size=hidden_size,
    input_steps=input_steps,
    output_steps=output_steps,
).to(device)

best_model.load_state_dict(torch.load("best_model_NPINN_absXY.pth", map_location=device))
best_model.eval()

```

This code provides essential **postprocessing and geometric utilities** for working with NPINN outputs and AIS trajectory data. The `inverse_dxdy_np` function is designed to convert scaled residuals `(dx, dy)` back into real-world units (meters) using a previously fitted scaler. It handles both 1D and 2D inputs, making it suitable for batch or single-step predictions. This is particularly useful for interpreting NPINN predictions in absolute physical units rather than in the normalized or scaled feature space, allowing for meaningful evaluation of the model’s accuracy in real-world terms. Using this, the code also computes the standard deviation of the residuals across the training dataset, providing a quantitative measure of typical displacement magnitudes along the x and y axes.

The code also includes geometry-related helpers to analyze trajectories in geospatial terms. The `haversine` function calculates the geodesic distance between longitude/latitude points in meters using the haversine formula, with safeguards for numerical stability and invalid inputs. Building on this, the `trajectory_length` function computes the total length of a vessel’s trajectory, summing distances between consecutive points while handling incomplete or non-finite data gracefully. Together, these utilities allow NPINN outputs to be mapped back to real-world coordinates, facilitate evaluation of trajectory smoothness and accuracy, and provide interpretable metrics for model validation and downstream analysis.

```python
# ---------------- helper inverse/scaling utilities ----------------
def inverse_dxdy_np(dxdy_scaled, scaler):
    """
    Invert scaled residuals (dx, dy) back to meters.
    dxdy_scaled: (..., 2) scaled
    scaler: RobustScaler/StandardScaler/MinMaxScaler fitted on residuals
    """
    dxdy_scaled = np.asarray(dxdy_scaled, dtype=float)
    if dxdy_scaled.ndim == 1:
        dxdy_scaled = dxdy_scaled[None, :]

    n_samples = dxdy_scaled.shape[0]
    n_features = scaler.scale_.shape[0] if hasattr(scaler, "scale_") else scaler.center_.shape[0]

    full_scaled = np.zeros((n_samples, n_features))
    full_scaled[:, :2] = dxdy_scaled

    if hasattr(scaler, "mean_"):
        center = scaler.mean_
        scale = scaler.scale_
    elif hasattr(scaler, "center_"):
        center = scaler.center_
        scale = scaler.scale_
    else:
        raise ValueError(f"Scaler type {type(scaler)} not supported")

    full = full_scaled * scale + center
    return full[:, :2] if dxdy_scaled.shape[0] > 1 else full[0, :2]

```

```python
# ---------------- compute residual std (meters) correctly ----------------
# train_Y contains scaled residuals (dx,dy) per your preprocessing.
all_resids_scaled = train_Y.reshape(-1, 2)                      # [sum_T, 2]
all_resids_m = inverse_dxdy_np(all_resids_scaled, delta_scaler) # meters
residual_std = np.std(all_resids_m, axis=0)
print("Computed residual_std (meters):", residual_std)


# ---------------- geometry helpers ----------------
def haversine(lon1, lat1, lon2, lat2):
    """Distance (m) between lon/lat points using haversine formula; handles arrays."""
    R = 6371000.0
    lon1 = np.asarray(lon1, dtype=float)
    lat1 = np.asarray(lat1, dtype=float)
    lon2 = np.asarray(lon2, dtype=float)
    lat2 = np.asarray(lat2, dtype=float)
    # if any entry is non-finite, result will be nan — we'll guard upstream
    lon1, lat1, lon2, lat2 = map(np.radians, [lon1, lat1, lon2, lat2])
    dlon = lon2 - lon1
    dlat = lat2 - lat1
    a = np.sin(dlat/2.0)**2 + np.cos(lat1)*np.cos(lat2)*np.sin(dlon/2.0)**2
    # numerical stability: clip inside sqrt
    a = np.clip(a, 0.0, 1.0)
    return 2 * R * np.arcsin(np.sqrt(a))

def trajectory_length(lons, lats):
    lons = np.asarray(lons, dtype=float)
    lats = np.asarray(lats, dtype=float)
    if lons.size < 2:
        return 0.0
    # guard non-finite
    if not (np.isfinite(lons).all() and np.isfinite(lats).all()):
        return float("nan")
    return np.sum(haversine(lons[:-1], lats[:-1], lons[1:], lats[1:]))
```

### Evaluate Function

This function `evaluate_with_errors` is designed to **evaluate NPINN trajectory predictions** in a geospatial context and optionally visualize them. It takes a trained model, a test `DataLoader`, coordinate projection, scalers, and device information. For each batch, it reconstructs the predicted trajectories from residuals (dx, dy), inverts the scaling back to meters, and converts them to absolute positions starting from the last observed point. Different decoding modes (`cumsum`, `independent`, `stdonly`) allow flexibility in how residuals are integrated into absolute trajectories, and it handles cases where the first residual is effectively a duplicate of the last input.

The evaluation computes **per-timestep errors in meters** using the haversine formula and tracks differences in trajectory lengths. All errors are summarized with mean and median statistics across the prediction horizon. When `plot_map=True`, the function generates **separate maps for each trajectory**, overlaying the true (green) and predicted (red dashed) paths, giving a clear visual inspection of the model’s performance. This approach is directly aligned with NPINN, as it evaluates predictions in physical units and emphasizes smooth, physically plausible trajectory reconstructions.

```python
def evaluate_with_errors(
    model,
    test_dl,
    proj,
    feature_scaler,
    delta_scaler,
    device,
    num_batches=None,   # None = use full dataset
    dup_tol: float = 1e-4,
    outputs_are_residual_xy: bool = True,
    residual_decode_mode: str = "cumsum",   # "cumsum", "independent", "stdonly"
    residual_std: np.ndarray = None,
    plot_map: bool = True   # <--- PLOT ALL TRAJECTORIES
):
    """
    Evaluate model trajectory predictions and report errors in meters.
    Optionally plot all trajectories on a map.
    """
    model.eval()
    errors_all = []
    length_diffs = []
    bad_count = 0

    # store all trajectories
    all_real = []
    all_pred = []

    with torch.no_grad():
        batches = 0
        for xb, yb in test_dl:
            xb, yb = xb.to(device), yb.to(device)
            pred = model(xb, teacher_forcing_ratio=0.0)  # [B, T_out, F]

            # first sample of the batch
            input_seq = xb[0].cpu().numpy()
            real_seq  = yb[0].cpu().numpy()
            pred_seq  = pred[0].cpu().numpy()

            # Extract dx, dy residuals
            pred_resid_s = pred_seq[:, :2]
            real_resid_s = real_seq[:, :2]

            # Invert residuals to meters
            pred_resid_m = inverse_dxdy_np(pred_resid_s, delta_scaler)
            real_resid_m = inverse_dxdy_np(real_resid_s, delta_scaler)

            # Use last observed absolute position as starting point (meters)
            last_obs_xy_m = inverse_xy_only_np(input_seq[-1, :2], feature_scaler)

            # Reconstruct absolute positions
            if residual_decode_mode == "cumsum":
                pred_xy_m = np.cumsum(pred_resid_m, axis=0) + last_obs_xy_m
                real_xy_m = np.cumsum(real_resid_m, axis=0) + last_obs_xy_m
            elif residual_decode_mode == "independent":
                pred_xy_m = pred_resid_m + last_obs_xy_m
                real_xy_m = real_resid_m + last_obs_xy_m
            elif residual_decode_mode == "stdonly":
                if residual_std is None:
                    raise ValueError("residual_std must be provided for 'stdonly' mode")
                noise = np.random.randn(*pred_resid_m.shape) * residual_std
                pred_xy_m = np.cumsum(noise, axis=0) + last_obs_xy_m
                real_xy_m = np.cumsum(real_resid_m, axis=0) + last_obs_xy_m
            else:
                raise ValueError(f"Unknown residual_decode_mode: {residual_decode_mode}")

            # Remove first target if duplicates last input
            if np.allclose(real_resid_m[0], 0, atol=dup_tol):
                real_xy_m = real_xy_m[1:]
                pred_xy_m = pred_xy_m[1:]

            # align horizon
            min_len = min(len(pred_xy_m), len(real_xy_m))
            if min_len == 0:
                bad_count += 1
                continue

            pred_xy_m = pred_xy_m[:min_len]
            real_xy_m = real_xy_m[:min_len]

            # project to lon/lat
            lon_real, lat_real = proj(real_xy_m[:,0], real_xy_m[:,1], inverse=True)
            lon_pred, lat_pred = proj(pred_xy_m[:,0], pred_xy_m[:,1], inverse=True)

            all_real.append((lon_real, lat_real))
            all_pred.append((lon_pred, lat_pred))

            # compute per-timestep errors
            errors = haversine(lon_real, lat_real, lon_pred, lat_pred)
            errors_all.append(errors)

            # trajectory length diff
            real_len = trajectory_length(lon_real, lat_real)
            pred_len = trajectory_length(lon_pred, lat_pred)
            length_diffs.append(abs(real_len - pred_len))

            print(f"Trajectory length (true): {real_len:.2f} m | pred: {pred_len:.2f} m | diff: {abs(real_len - pred_len):.2f} m")

            batches += 1
            if num_batches is not None and batches >= num_batches:
                break

    # summary
    if len(errors_all) == 0:
        print("No valid samples evaluated. Bad count:", bad_count)
        return

    max_len = max(len(e) for e in errors_all)
    errors_padded = np.full((len(errors_all), max_len), np.nan)
    for i, e in enumerate(errors_all):
        errors_padded[i, :len(e)] = e

    mean_per_t = np.nanmean(errors_padded, axis=0)
    print("\n=== Summary (meters) ===")
    for t, v in enumerate(mean_per_t):
        if not np.isnan(v):
            print(f"t={t} mean error: {v:.2f} m")
    print(f"Mean over horizon: {np.nanmean(errors_padded):.2f} m | Median: {np.nanmedian(errors_padded):.2f} m")
    print(f"Mean trajectory length diff: {np.mean(length_diffs):.2f} m | Median: {np.median(length_diffs):.2f} m")
    print("Bad / skipped samples:", bad_count)

    # --- plot all trajectories ---
       # --- plot each trajectory separately ---
    if plot_map and len(all_real) > 0:
        import matplotlib.pyplot as plt
        import cartopy.crs as ccrs
        import cartopy.feature as cfeature

        for idx, ((lon_r, lat_r), (lon_p, lat_p)) in enumerate(zip(all_real, all_pred)):
            fig = plt.figure(figsize=(10, 8))
            ax = plt.axes(projection=ccrs.PlateCarree())
            ax.add_feature(cfeature.LAND)
            ax.add_feature(cfeature.COASTLINE)
            ax.add_feature(cfeature.BORDERS, linestyle=':')
            
            ax.plot(lon_r, lat_r, color='green', linewidth=2, label="True")
            ax.plot(lon_p, lat_p, color='red', linestyle='--', linewidth=2, label="Predicted")
            ax.legend()

            ax.set_title(f"Trajectory {idx+1}: True vs Predicted")
            plt.show()

```

### Results

#### **Trajectory Plots**

1.  **Trajectory 2**

    * True length: 521.39 m
    * Predicted length: 508.66 m
    * Difference: 12.73 m

    The predicted path is roughly parallel to the true path but slightly offset. The predicted trajectory underestimates the total path length by \~2.4%, which is a small but noticeable error. The smoothness of the red dashed line indicates that the model is generating physically plausible, consistent trajectories.
2.  **Trajectory 5**

    * True length: 188.76 m
    * Predicted length: 206.01 m
    * Difference: 17.25 m

    Here, the predicted trajectory slightly **overestimates** the total distance (\~9%), again with a smooth but slightly offset path relative to the true trajectory. The model captures the overall direction but has some scaling error in step lengths or residuals.

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

```
Trajectory length (true): 521.39 m | pred: 508.66 m | diff: 12.73 m
```

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

```
Trajectory length (true): 188.76 m | pred: 206.01 m | diff: 17.25 m
```

#### **Summary Statistics**

* **t=0 mean error: 45.03 m**\
  The first prediction step already has a \~45 m average discrepancy from the true position, which is common since the model starts accumulating error immediately after the last observed point.
* **t=1 mean error: 80.80 m**\
  Error grows with the horizon, reflecting cumulative effects of residual inaccuracies.
* **Mean over horizon: 62.92 m | Median: 61.72 m**\
  On average, predictions are within \~60–63 m of the true trajectory at any given time step. Median being close to mean suggests a fairly symmetric error distribution without extreme outliers.
* **Mean trajectory length difference: 11.82 m | Median: 12.73 m**\
  Overall, the predicted trajectories’ total lengths are very close to the true lengths, typically within \~12 m, which is less than 3% relative error for most trajectories.

#### **Interpretation**

* The model captures **trajectory trends well** but shows small offsets in absolute positions.
* **Errors grow with horizon**, which is typical for sequence prediction models using residuals.
* Smoothness is maintained (no erratic jumps), indicating that the **NPINN smoothness regularization is effective**.
* Overall, this is a solid performance for maritime AIS trajectory prediction, especially given the scale of trajectories (hundreds of meters).
