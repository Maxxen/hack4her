# Table of Contents
- [Table of Contents](#table-of-contents)
- [Useful links](#useful-links)
- [General Tips](#general-tips)
- [Datasets](#datasets)
  - [Amsterdam Pedestrian Road Network](#amsterdam-pedestrian-road-network)
    - [Road Network Nodes](#road-network-nodes)
    - [Road Network Edges](#road-network-edges)
  - [Amsterdam Public Light Locations](#amsterdam-public-light-locations)
  - [Amsterdam Wijken (Districts)](#amsterdam-wijken-districts)
    - [Including Water](#including-water)
    - [Excluding Water](#excluding-water)
  - [Amsterdam Buurten (Neighborhoods)](#amsterdam-buurten-neighborhoods)
    - [Including Water](#including-water-1)
    - [Excluding Water](#excluding-water-1)
  - [Amsterdam A10 Ring Polygon](#amsterdam-a10-ring-polygon)
  - [Amsterdam Safety Statistics](#amsterdam-safety-statistics)
  - [Overture Maps Data](#overture-maps-data)


# Useful links

- [DuckDB Documentation](https://duckdb.org/docs/)
- [DuckDB Spatial Extension Documentation](https://duckdb.org/docs/sql/extensions/spatial)
- [DuckDB Spatial Full Function Reference](https://github.com/duckdb/duckdb_spatial/blob/main/docs/functions.md)
- [Dr. Qiusheng Wu's Geospatial Data Science with DuckDB + Python (GEOG-414)](https://geog-414.gishub.org/book/duckdb/01_duckdb_intro.html). This is a great in-depth resource for learning how to use DuckDB for both spatial and non-spatial SQL.
- [Accessing Overture Maps Data with DuckDB](https://docs.overturemaps.org/getting-data/duckdb/)

# General Tips

- __Routing__ DuckDB cannot help you with routing if you wish to perform shortest-path calculations. You can use DuckDB to enrich and/or create a dataset for your road network but you most likely need a graph library for the path calculations. I would recommend the `networkx` library if you are using Python. Alternatively you can maybe avoid routing and instead focus on other aspects of the project by e.g. letting users supply their own routes.

- __Offline vs Online__. Its worth thinking about what kind of data you can pre-compute and store in e.g. a DuckDB database and what calculations you actually need to compute on-the-fly from user input.

- __Aggregating is your friend__.
    Instead of counting e.g. all the restuarants within 100 meters for all the road segments, pre-compute the restuarant density in each neighborhood or "cell" or some other arbitrary partitioning of the area (like a square 250x250m grid covering all of amsterdam). Then you only have to join the segments with the cells instead, which might reduce the amount of work the join has to do considerably at the cost of statistical fidelity.

- __Focus on Area of Interest__ Consider limiting the area of interest further (e.g only within the A10 ring) if performance is an issue. You can get away with a lot of simplifications and brute-force solutions if the dataset is small enough.

- __Always know what coordinate system your geometries are in__. Units of measurements (like distance and area) can get messy/inaccurate or you can end up with very unexpected results if you try to e.g. join two datasets in different coordinate systems. lon/lat vs lat/lon is also particularly easy to get wrong, in which case `ST_FlipCoordinates` can help.

# Datasets

This list contain a number of interesting useful datasets covering the Amsterdam area. If you wish to do network analysis and calculate the shortest route given two arbitrary points you will most likely need to use the [pedestrian road network](#amsterdam-pedestrian-road-network) as a base to create your graph from, but you can use the other datasets to enrich your cost model with additional information to facilitate the calculation of not just shortes, but the the _safest_ route as well. 

You are of course free to use any other datasets you find interesting or relevant to your project. The Amsterdam municipality in general has a ton of interesting open data available on their [open data portal](https://maps.amsterdam.nl/open_geodata/) (where most of these are taken from).

## Amsterdam Pedestrian Road Network

This dataset models the pedestrian road network within Amsterdams A10 ring as a graph consisting of __edges__ and __nodes__. 

### Road Network Nodes

[Download link (GeoJSON Lng/Lat)](https://blobs.duckdb.org/ams_walk_nodes.geojson)

__Schema__:

| Column Name | Column Type | Description                                                              |
| ----------- | ----------- | ------------------------------------------------------------------------ |
| id          | INTEGER     | `id` uniquely identifying this node                                      |
| geom        | GEOMETRY    | The geometry of this node, always a `POINT` in Longitude/Latitude coordinates |


### Road Network Edges

[Download link (GeoJSON Lng/Lat)](https://blobs.duckdb.org/ams_walk_edges.geojson)

__Schema__:

| Column Name | Column Type | Description                                                                                                                                   |
| ----------- | ----------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| id          | INTEGER     | `id` uniquely identifying this edge                                                                                                           |
| u           | INTEGER     | `id` of the "start" node for this edge                                                                                                        |
| v           | INTEGER     | `id` of the "end" node for this edge                                                                                                          |
| key         | INTEGER     | In case there are multiple edges with the same `u` and `v` (connecting the same two nodes), the `key` can be used to distinguish between them |
| highway     | VARCHAR     | The type of road this segment represents (e.g. `footway`, `residential`, `service`, `unclassified`)                                           |
| name        | VARCHAR     | The name of the street this segment lies on (if the street has a name)                                                                       |
| bridge      | BOOLEAN     | If this segment is on a bridge                                                                                                                |
| tunnel      | BOOLEAN     | If this segement is in a tunnel                                                                                                               |
| geom        | GEOMETRY    | The geometry of this segment, always a `LINESTRING` in Longitude/Latitude coordinates                                                         |
| length      | DOUBLE      | The length of this segment in meters                                                                                                          |


```sql
-- Import both the edges and nodes into DuckDB
LOAD spatial;
LOAD httpfs;
CREATE TABLE ams_walk_nodes AS SELECT * FROM st_read('https://blobs.duckdb.org/ams_walk_nodes.geojson');
CREATE TABLE ams_walk_edges AS SELECT * FROM st_read('https://blobs.duckdb.org/ams_walk_edges.geojson');
```

## Amsterdam Public Light Locations

Suggestions:
 - This dataset contains 146769 points, but you can reduce it substantially by filtering out points lying outside your area of interest (for example, [the A10 ring](#amsterdam-a10-ring-polygon)).
 - Some entries have a `Straatnaam` field specifying the name of the street this light is installed in. Do these map nicely to the street names in the [road network](#amsterdam-pedestrian-road-network) or is it better to join the two datasets spatially to calculate the light level of a street segment?
 - Some of the entries in this dataset contains `Lamp_Wattage` and `Lamp_Lumen` that could be used either individually or together to model the strength of the light, but for many entries both of these values are 0 (perhaps missing?). However, almost all have a `Tiepe` field specifying the type of light. Is there a correlation between lamp strength and type? If so, could you come up with a plausible "default" strength to use instead for the lights with 0 Lumen or Wattage?

[Information Page](https://maps.amsterdam.nl/open_geodata/?k=510)

[Download link (GeoJSON Lng/Lat)](https://maps.amsterdam.nl/open_geodata/geojson_lnglat.php?KAARTLAAG=LICHTPUNTEN&THEMA=lichtpunten)

```sql
-- Import into DuckDB
LOAD spatial;
LOAD httpfs;
CREATE TABLE ams_lights AS SELECT * FROM st_read('https://maps.amsterdam.nl/open_geodata/geojson_lnglat.php?KAARTLAAG=LICHTPUNTEN&THEMA=lichtpunten');
```

## Amsterdam Wijken (Districts)

This dataset contains the polygons of the different districts in Amsterdam. It can be used to aggregate data on a district level. Note that a district can contain multiple neighborhoods.

You can use this [online map](https://maps.amsterdam.nl/gebiedsindeling/) to see the different districts and neighborhoods in Amsterdam.

### Including Water

[Information Page](https://maps.amsterdam.nl/open_geodata/?k=200)

[Download Page (GeoJSON Lng/Lat)](https://maps.amsterdam.nl/open_geodata/geojson_lnglat.php?KAARTLAAG=INDELING_WIJK&THEMA=gebiedsindeling)

```sql
-- Import into DuckDB
LOAD spatial;
LOAD httpfs;
CREATE TABLE ams_wijken AS SELECT * FROM st_read('https://maps.amsterdam.nl/open_geodata/geojson_lnglat.php?KAARTLAAG=INDELING_WIJK&THEMA=gebiedsindeling');
```

### Excluding Water

This variant of the dataset excludes areas covered by water from the district polygons. This means that some districts end up being represented by `MULTIPOLYGON`'s as they are internally divided by e.g. large canals or docks.

[Information Page](https://maps.amsterdam.nl/open_geodata/?k=201)

[Download Page (GeoJSON Lng/Lat)](https://maps.amsterdam.nl/open_geodata/geojson_lnglat.php?KAARTLAAG=INDELING_WIJK_EXWATER&THEMA=gebiedsindeling)

```sql
-- Import into DuckDB
LOAD spatial;
LOAD httpfs;
CREATE TABLE ams_wijken AS SELECT * FROM st_read('https://maps.amsterdam.nl/open_geodata/geojson_lnglat.php?KAARTLAAG=INDELING_WIJK_EXWATER&THEMA=gebiedsindeling');
```

## Amsterdam Buurten (Neighborhoods)

This dataset contains the polygons of the different neighborhoods in Amsterdam. It can be used to aggregate data on a neighborhood level. Note that a neighborhood always belongs to a single district (Wijken).

You can use this [online map](https://maps.amsterdam.nl/gebiedsindeling/) to see the different districts and neighborhoods in Amsterdam.

### Including Water

[Information Page](https://maps.amsterdam.nl/open_geodata/?k=198)

[Download Page (GeoJSON Lng/Lat)](https://maps.amsterdam.nl/open_geodata/geojson_lnglat.php?KAARTLAAG=INDELING_BUURT&THEMA=gebiedsindeling)

```sql
-- Import into DuckDB
LOAD spatial;
LOAD httpfs;
CREATE TABLE ams_buurten AS SELECT * FROM st_read('https://maps.amsterdam.nl/open_geodata/geojson_lnglat.php?KAARTLAAG=INDELING_BUURT&THEMA=gebiedsindeling');
```

### Excluding Water

[Information Page](https://maps.amsterdam.nl/open_geodata/?k=199)

[Download Page (GeoJSON Lng/Lat)](https://maps.amsterdam.nl/open_geodata/geojson_lnglat.php?KAARTLAAG=INDELING_BUURT_EXWATER&THEMA=gebiedsindeling)

```sql
-- Import into DuckDB
LOAD spatial;
LOAD httpfs;
CREATE TABLE ams_buurten AS SELECT * FROM st_read('https://maps.amsterdam.nl/open_geodata/geojson_lnglat.php?KAARTLAAG=INDELING_BUURT_EXWATER&THEMA=gebiedsindeling');
```

## Amsterdam A10 Ring Polygon

This dataset just contains a single large polygon covering the are encircled by the A10 highway going around the center of amsterdam. This can be used to "clip" and filter other larger spatial datasets so that they only contain geometries within the A10 ring.

[Download link (GeoJSON Lng/Lat)](https://blobs.duckdb.org/ams_a10.geojson)

```sql
-- Import into DuckDB
LOAD spatial;
LOAD httpfs;
CREATE TABLE ams_a10 AS SELECT * FROM st_read('https://blobs.duckdb.org/ams_a10.geojson');
```

## Amsterdam Safety Statistics

This dataset contain multiple files with different types of safety statistics (like crime rate, feeling of victimization) for the different districts (Wijken) of Amsterdam for different times of the year.
Interestingly enough there is not one single district that is the "safest" in all categories (except maybe central which is almost always "bad"), so you might want to consider how you want to aggregate these statistics to create a single "safety" metric for each district, or just focus on a single category.

[Main page (Click on the map to select district)](https://onderzoek.amsterdam.nl/interactief/dashboard-veiligheid?gebied=1&index=criminaliteit&meting=2023-3&indeling=wijk&subindex=totaal)

[There is also more data on this page (in XLSX format)](https://onderzoek.amsterdam.nl/dataset/cijfers-veiligheidsindex)

Suggestions:
  - If you want to join these with the Amsterdam Wijken polygons it might be easier to use the "code" (e.g. "AE", "AD") used to identify the wijken instead of the name in case there different casing/spelling.


## Overture Maps Data

Accessing overture maps data is slightly more complicated but it turns out that DuckDB still is one of the easiest ways to do it. 
You can find more information on how to access the data with DuckDB [here](https://docs.overturemaps.org/getting-data/duckdb/).

[Here is the list of the datasets available](https://docs.overturemaps.org/schema/reference/) in the overture maps dataset

Here might be a quick example of how to access just some road segment data roughly covering the amsterdam area

```sql
-- Scan amsterdam road segments from the overture maps dataset
CREATE OR REPLACE TABLE roads AS
    SELECT
        class,
        names.primary AS name,
        JSON_EXTRACT_STRING(road, '$.class') AS class,
        ST_GeomFromWKB(geometry) as geometry,
        id,
        connector_ids
    FROM read_parquet('s3://overturemaps-us-west-2/release/2024-05-16-beta.0/theme=transportation/type=segment/*')
    WHERE
        subtype = 'road'
        AND JSON_EXTRACT_STRING(road, '$.')
        AND bbox.xmin > 4.727554 AND bbox.xmax < 5.023499
        AND bbox.ymin > 52.276981 AND bbox.ymax < 52.441362;
```