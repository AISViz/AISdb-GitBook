---
icon: brain-circuit
---

# AISdb Made Easy: A No-Code Interface

The AISdb no-code interface offers a complete, visual way to process and analyze AIS data without writing a single line of code. Users can split trajectories by time, encode vessel tracks, discretize locations using H3, and detect vessel stops with adjustable parameters. The interface also supports adding environmental context such as bathymetry and weather layers. With customizable controls for gap duration, time segmentation, encoding distance and speed thresholds, and H3 resolution, the platform streamlines preprocessing while maintaining full analytical flexibility. Whether you’re preparing data for modeling, visualization, or research, every step—from segmentation to environmental enrichment—can be configured directly through an intuitive, click-based workflow.\
\
Below is the file you can just download and run locally to access the interface

{% file src="../.gitbook/assets/AIS-BOT 1.py" %}

### Interface

On the landing page you can just drag and drop your csv with AIS data, the assistant will automatically give you details about the csv, total rows, mean etc. .

<figure><img src="../.gitbook/assets/image (47).png" alt=""><figcaption></figcaption></figure>



#### Process&#x20;

**Steps -**&#x20;

1. `Split by time`: Breaks continuous vessel tracks into separate segments when there's a significant time gap in the data
2. `Encode track`: Applies distance and speed-based encoding to smooth and normalize vessel trajectories
3. `Discretize H3`: Converts geographic coordinates into H3 hexagonal grid cells for spatial analysis
4. `Detect stops`: Identifies vessel stopping periods based on speed thresholds and duration
5. `Add bathymetry`: Incorporates water depth data from GeoTIFF files along vessel routes
6. `Add weather`: Integrates weather data (wind, pressure, etc.) with vessel tracking information

**Gap minutes (default: 4320) -** Maximum allowed time gap between consecutive AIS points before splitting into new track segments

**Time split (days) (default: 15) -** Alternative way to split tracks by specifying the maximum duration in days

**Encode distance (m)** **(default: 200000) -** Maximum distance threshold in meters for track encoding/smoothing

**Encode speed (knots)** **(default: 50) -** Maximum speed threshold in knots for track encoding/filtering

**H3 resolution (default: 6) -** Resolution level for H3 hexagonal grid cells (0-15, higher means smaller cells)

**Bathymetry raster -** Path to GeoTIFF file containing water depth data

**Weather shortnames -** List of weather parameters to include (e.g., '10u,10v,msl' for wind components and sea level pressure)

**Distance split (m) (default: 30000) -** Distance threshold in meters for breaking tracks into segments

**Speed split (knots) (default: 30) -** Speed threshold in knots for track segmentation

#### Static Plot

The Static Plot feature leverages Matplotlib and Cartopy to create high-quality, publication-ready visualizations of AIS vessel tracks. It offers options for coastline overlays, multiple export formats (PNG/PDF), and precise geographic projections, making it ideal for scientific publications and detailed analysis of maritime traffic patterns.

#### Plotly OSM

Plotly OSM (OpenStreetMap) provides an interactive visualization experience, combining vessel tracking data with detailed map layers. This feature enables dynamic exploration of maritime routes with zoom capabilities, hover information, and the ability to toggle between line and marker modes. The interactive HTML export option makes it perfect for web-based presentations and sharing insights.

#### Explain

The Explain feature harnesses LangChain and Google's Generative AI to provide natural language insights about your AIS data. Simply ask questions about your dataset, and receive detailed explanations about patterns, anomalies, and statistics, making complex maritime data analysis accessible to non-technical stakeholders.

#### Gemini

The Gemini integration takes maritime data visualization to the next level by combining Google's advanced AI model with visual context. It can analyze both your data and generated plots simultaneously, offering comprehensive insights, pattern recognition, and detailed explanations of maritime behavior that might not be immediately apparent to human observers.
