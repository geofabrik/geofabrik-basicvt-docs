+++
draft= false
title = "Creating Shortbread Vector Tiles"
description = "This page describes how to generate the vector tiles in the Shortbread schema"
+++

Creating Shortbread vetor tiles consists of a couple of steps. This page provides an overview about
the tile generation process. It consists of the following steps.

* Install dependencies
* Clone schema repository
* Download simplified ocean polygons
* Generate vector tile set using Tilemaker

## Dependencies

In order to build vector tiles, you have to provide the following dependencies:

* Curl
* Unzip
* GDAL
* Install Tilemaker

Ubuntu/Debian: `apt install curl unzip gdal-bin`

## Tilemaker Installation

The Shortbread vector tile schema requires a special feature of Tilemaker (sorting features in a
layer by a float number). Therefore, Tilemaker has to be compiled with `FLOAT_Z_ORDER` set to any value.

Clone and compile Tilemaker:

```sh
git clone --branch v2.4.0 https://github.com/systemed/tilemaker.git
cd tilemaker
make CONFIG=-DFLOAT_Z_ORDER
```

Installation of Tilemaker is described in the [Tilemaker Readme](https://github.com/geofabrik/tilemaker/#installing) as well.


## Clone Schema Repository

Clone the Git repository of the vector tile schema from Github:

```sh
mkdir shortbread-tilemaker
git clone https://github.com/geofabrik/shortbread-tilemaker.git shortbread-tilemaker
```

## Download Ocean Shape Files

The vector tile schema retrieves ocean polygons fro pre-processed ocean polygon shape files by osmdata.openstreetmap.de.
A script is provided to download and update them. It will write the shape files to the subdirectory `data`.

Tilemaker requires shape files in geographic coordinates (EPSG:4326) but the simplified shape file is provided
in Web Mercator (EPSG:3857) only. Therefore, GDAL's *ogr2ogr* is used to transform the shape file.

```sh
cd shortbread-tilemaker
./get-shapefiles.sh
```

## Download OpenStreetMap Raw Data

Tilemaker needs raw OpenStreetMap data in `.osm.pbf` format as input. You can download it from

* regional extracts from [download.geofabrik.de](https://download.geofabrik.de)
* regional extracts from [download.openstreetmap.fr](https://download.openstreetmap.fr/extracts/)
* latest weekly planet PBF file from [planet.openstreetmap.org](https://planet.openstreetmap.org/) or any of its [mirrors](https://wiki.openstreetmap.org/wiki/Planet.osm#Planet.osm_mirrors)

Please try with a small extract first before you try to load the complete world.


## Generate Vector Tileset

Generating vector tiles seems to be pretty simple but there are some show stoppers to keep in mind:

* If you write vector tiles as individual files to disk, mind that they need lots of inodes (about
  300 million). Reformat the target device or loop-mount a file.
* Tilemaker benefits from a cache on disk. It should be located on a very fast device (SSD/NVMe,
  no hard disk or network drive).
* The .osm.pbf file contains a bounding box entry in its header (this is not true for the planet),
  Tilemaker will use it as a spatial filter if you do not provide a bounding box on command line.
  The bounding box is applied on all input sources (OSM data and shape file) and decides which
  vector tiles will be created at all. Cache size depends on the number of tiles.

Run Tilemaker (this command should be executed from the `shortbread-tilemaker` directory):

```sh
tilemaker --bbox -180,-86,180,86 --input planet-latest.osm.pbf --store tilemaker-cache.dat --config config.json --process process.lua --output output_directory/
```

In February 2022, a server with a AMD EPYC 7452 32-Core Processor (2.35–3.35 GHz), 512 GB RAM,
cache on NVMe and output to a loop-mounted hard disk drive (RAID 1) took 16:15 hours and needed up
to 358 GB RAM.

## Troubleshooting

### "`z_order` is limited to 1 byte signed integer"

Error message:

```
lua runtime error: z_order is limited to 1 byte signed integer.
terminate called after throwing an instance of 'kaguya::LuaTypeMismatch'
  what():  type mismatch!!

```

Reason: Your version of Tilemake was compiled without the flag `FLOAT_Z_ORDER`.

Solution: Recompile Tilemaker using `make clean && make CONFIG=-DFLOAT_Z_ORDER`.
