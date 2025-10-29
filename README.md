# Wrangling Geospatial Data with GeoPandas and Apache Sedona

This repository contains examples and tutorials for processing and analyzing geospatial data using **GeoPandas** (for efficient in-memory processing on a single machine) and **Apache Sedona** (for distributed computing on Apache Spark/AWS Glue).

## Why These Tools?

Modern geospatial datasets consist of complex geometric vectors (points, lines, polygons) and dense raster grids. Traditional low-level libraries like Shapely, Fiona, and GEOS make it hard to combine spatial data with standard data science workflows.

- **GeoPandas** wraps complex geometric operations into familiar pandas DataFrames, enabling sophisticated spatial operations (spatial joins, buffering, projections) with simple Python code. Integrates seamlessly with NumPy, Matplotlib, and Scikit-learn.

- **Apache Sedona** brings distributed computing to spatial datasets. Without Sedona, writing a spatial join on billions of records in raw Spark would require manually partitioning data, building spatial indexes, and managing join logic across a cluster. Sedona provides spatial DataFrames, spatial SQL functions, and automatic spatial indexing.

## When to Use GeoPandas vs Apache Sedona

| Feature | GeoPandas | Apache Sedona |
|---------|-----------|---------------|
| **Architecture** | Single machine, in-memory | Distributed cluster (Spark/Flink/Snowflake) |
| **Best For** | Interactive exploration, POCs, visualization | Production big data workloads (petabytes) |
| **Dataset Size** | Small to medium (< few GB) | Large to massive (unlimited) |
| **Speed** | Fast for small data | Scales linearly with cluster size |

## Setup Instructions

### GeoPandas (Local Setup)

1. **Install UV** (fast Python package manager):
   ```bash
   # Using pip
   pip install uv
   
   # Or using curl (Unix/macOS)
   curl -LsSf https://astral.sh/uv/install.sh | sh
   ```

2. **Clone this repository**:
   ```bash
   git clone <your-repo-url>
   cd geo_pandas_example
   ```

3. **Install dependencies** (Python 3.12):
   ```bash
   uv sync
   ```
   This creates a `.venv` virtual environment with all required packages (geopandas, jupyter, matplotlib, folium, etc.).

4. **Activate environment and run Jupyter**:
   ```bash
   # Activate virtual environment
   source .venv/bin/activate  # macOS/Linux
   .venv\Scripts\activate     # Windows
   
   # Launch Jupyter
   jupyter notebook
   
   # Or use uv run (no activation needed)
   uv run jupyter notebook
   ```

5. **VSCode Users**: Open any `.ipynb` file and select the `.venv/bin/python` interpreter from the kernel selector.

### Apache Sedona (AWS Glue Setup)

1. **Download required JAR files**:
   - [Sedona Spark JAR](https://repo1.maven.org/maven2/org/apache/sedona/sedona-spark-shaded-3.3_2.12/1.7.0/sedona-spark-shaded-3.3_2.12-1.7.0.jar) (for Spark 3.3, Scala 2.12, Sedona 1.7)
   - [GeoTools Wrapper JAR](https://repo1.maven.org/maven2/org/datasyslab/geotools-wrapper/1.7.1-28.5/geotools-wrapper-1.7.1-28.5.jar) (handles CRS transformations)

2. **Upload JARs to S3**:
   ```bash
   aws s3 cp sedona-spark-shaded-3.3_2.12-1.7.0.jar s3://your-bucket-name/jars/
   aws s3 cp geotools-wrapper-1.7.1-28.5.jar s3://your-bucket-name/jars/
   ```

3. **Configure AWS Glue Notebook** (add these magics at the top):
   ```python
   %idle_timeout 2880
   %glue_version 4.0
   %worker_type G.1X
   %number_of_workers 2
   
   # Point to your JARs
   %extra_jars s3://your-bucket-name/jars/sedona-spark-shaded-3.3_2.12-1.7.0.jar,s3://your-bucket-name/jars/geotools-wrapper-1.7.1-28.5.jar
   %additional_python_modules apache-sedona==1.7.0
   ```

**Note**: If your security group allows outbound HTTPS and has internet gateway access, you can point directly to Maven URLs instead of S3.

## Notebooks Overview

### GeoPandas Notebooks (`geopandas/`)

#### `01_basic_geopandas.ipynb` - Fundamentals
- Creating GeoDataFrames from scratch (cities with Point geometries)
- Loading real-world datasets (Natural Earth countries)
- CRS transformations (EPSG:4326 → projected systems)
- Area calculations and centroids
- Reading/writing geospatial formats (GeoJSON, Shapefiles)
- Basic visualization with matplotlib

**Key operations**: `Point()`, `to_crs()`, `geometry.area`, `geometry.centroid`, `to_file()`, `read_file()`

#### `02_advanced_operations.ipynb` - Performance & Complex Operations
- **Spatial indexing**: R-tree for 10x+ speed improvements on large datasets
- **Dissolve**: Merging geometries by attributes (like SQL GROUP BY for shapes)
- **Vectorized vs iterative**: Performance comparison showing 33x speedup
- **Geometric transformations**: Simplification, envelopes, affine transformations
- **Complex operations**: Union, intersection, difference on polygons

**Key operations**: `.sindex`, `.dissolve()`, `.simplify()`, `.buffer()`, vectorized `.area`

#### `03_energy_grid_mapping.ipynb` - Power Grid Analysis
- Creating synthetic power grid infrastructure (25 substations, 5 power plants, transmission lines)
- Different power generation types (coal, solar, wind, nuclear, hydro)
- **Service area analysis**: Buffer-based coverage zones
- **Proximity analysis**: Finding nearest substations to demand points
- **Coverage gaps**: Identifying underserved areas
- **Interactive maps**: Folium visualization with multiple layers
- **Network statistics**: Grid capacity, coverage percentages, transmission corridors

**Use cases**: Infrastructure planning, coverage optimization, grid reliability, expansion planning

### Apache Sedona Notebook (`apache_sedona/`)

#### `apache_sedona_example.ipynb` - Distributed Geospatial Processing
Same power grid scenario as GeoPandas example but demonstrates **distributed computing at scale**:

- **Setup**: Sedona configuration in AWS Glue (Spark 3.3)
- **Spatial SQL**: Using `ST_*` functions (ST_Distance, ST_Within, ST_Transform, ST_Buffer)
- **Distributed operations**:
  - Nearest neighbor search with automatic spatial indexing
  - Service area coverage analysis (ST_Within)
  - Gap analysis with ST_Union_Aggr (removes overlaps)
  - Transmission corridor buffering
- **Performance optimizations**: Spatial partitioning, broadcast joins, R-tree indexing
- **Scalability demos**: Processing that would handle billions of records
- **Exporting to S3**: Writing results as Parquet/GeoParquet

**Key differences from GeoPandas**: Processes data in parallel across Spark cluster, handles datasets larger than memory, optimized for petabyte-scale workloads

## Quick Reference

### GeoPandas Essential Operations
```python
import geopandas as gpd
from shapely.geometry import Point

# Create GeoDataFrame
gdf = gpd.GeoDataFrame({'name': ['NYC'], 'geometry': [Point(-74, 40)]}, crs='EPSG:4326')

# CRS transformation
gdf_projected = gdf.to_crs('EPSG:3857')

# Spatial operations
gdf['buffer'] = gdf.geometry.buffer(1000)  # 1km buffer
gdf['area_km2'] = gdf.geometry.area / 1_000_000

# Spatial join
result = gpd.sjoin(points_gdf, polygons_gdf, predicate='within')
```

### Apache Sedona Essential Operations
```python
from sedona.spark import *

# Create Sedona context
config = SedonaContext.builder().getOrCreate()
sedona = SedonaContext.create(config)

# Spatial SQL queries
sedona.sql("""
    SELECT id, ST_Distance(a.geom, b.geom) as distance
    FROM points_a a, points_b b
    WHERE ST_Distance(a.geom, b.geom) < 1000
""")

# Common functions: ST_Buffer, ST_Within, ST_Transform, ST_Area, ST_Union_Aggr
```

## Troubleshooting

### GeoPandas Issues
- **CRS/PROJ errors (macOS)**: Install PROJ with `brew install proj`, then restart kernel
- **Import errors**: Ensure correct kernel selected (`.venv/bin/python`) or run `uv sync`
- **Slow performance**: Use spatial indexing (`.sindex`), vectorized operations, simplify geometries

### Apache Sedona Issues
- **JAR not found**: Verify S3 paths in `%extra_jars` magic, ensure JARs are uploaded
- **Version mismatch**: Use Spark 3.3 compatible JARs (Sedona 1.7.0, Scala 2.12)
- **Network errors**: Check security group allows outbound HTTPS, subnet has internet gateway

## Key Concepts

### GeoPandas Architecture
- **GeoSeries**: Manages geometric data (points, lines, polygons) using Shapely
- **GeoDataFrame**: Extended pandas DataFrame with one geometry column
- **Operations**: CRS transformations, spatial joins, buffering, area calculations
- **Best for**: Interactive analysis, visualization, datasets < few GB

### Apache Sedona Architecture  
- **Spatial DataFrames**: Distributed geometric data across Spark cluster
- **Spatial SQL**: Rich `ST_*` functions (ST_Distance, ST_Within, ST_Buffer, etc.)
- **Spatial Indexing**: Automatic R-tree/Quad-tree for optimized joins and queries
- **Best for**: Production workloads, big data (petabytes), high-performance processing

## Resources

### Documentation
- [GeoPandas](https://geopandas.org/) - Python geospatial library
- [Apache Sedona](https://sedona.apache.org/) - Distributed geospatial processing
- [Shapely](https://shapely.readthedocs.io/) - Geometric operations
- [UV Package Manager](https://github.com/astral-sh/uv) - Fast Python installer

### Maven Repository (for Sedona JARs)
- [Sedona Spark Packages](https://mvnrepository.com/artifact/org.apache.sedona)
- [GeoTools Wrapper](https://mvnrepository.com/artifact/org.datasyslab/geotools-wrapper)

---

**Happy Geospatial Wrangling!**