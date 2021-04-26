# jore4-hootenanny-experiment

Try to conflate maps with [Hootenanny](https://github.com/ngageoint/hootenanny).

## Workflow

### 1. Download a Digiroad extract

Here the 2021\_02 K extract was used which currently does not have a stable link as it is the latest data set:
<https://aineistot.vayla.fi/digiroad/latest/Maakuntajako_DIGIROAD_K_EUREF-FIN/UUSIMAA.zip>

Other [Digiroad](https://vayla.fi/en/transport-network/data/digiroad/data) data sets can be downloaded from [the open data portal of the Finnish Transport Infrastructure Agency](https://aineistot.vayla.fi/digiroad/).

### 2. Download an OpenStreetMap (OSM) extract

Here the OSM XML format was used:
<https://download.geofabrik.de/europe/finland-latest.osm.bz2>

Geofabrik offers other [daily extracts](https://download.geofabrik.de/europe/finland.html), as well.

Note that OSM is ODbL-licensed which will be the license of the end result of this experiment.
Switch to another tram network data source for another license.
We are not using the data result of this experiment in production so it does not matter.

### 3. Use osmosis to retain only the tram network from OSM

Install [osmosis](https://github.com/openstreetmap/osmosis) with your package manager.

Alternatively the repository seems to have a [Docker setup](https://github.com/openstreetmap/osmosis/blob/master/doc/development.md).

The following script was used to extract the tram network from the full OSM data set:
```bash
#!/bin/bash

set -Eeuxo pipefail

[ "$#" -ne 2 ] && {
  echo "Usage: $(basename "$0") INPUT_OSM_XML_PATH OUTPUT_OSM_XML_PATH" 1>&2
  exit 1
}

input="$1"
output="$2"

time osmosis \
  --read-pbf "${input}" \
  --log-progress \
  --tag-filter reject-relations \
  --tag-filter accept-ways railway=tram \
  --used-node \
  --write-xml "${output}"
```

### 4. Crop a small but relevant area from both data sources

To speed up finding suitable parameters for the conflation, crop a small but relevant area from both data sources.

Use the shapefile `DR_LINKKI_K.shp` from Digiroad.

Use QGIS to crop roughly the same rectangle from the Digiroad and the OSM tram data sets.

This could be automated.

### 5. Transform Digiroad shapefile into OSM XML format

The [HSLdevcom/ogr2osm](https://github.com/HSLdevcom/ogr2osm) branch `for-hootenanny-experiment` supports the data types in the Digiroad K data set and has a rudimentary Dockerfile.

Build the Docker image and run something similar to:
```sh
DOCKER_TAG="hsldevcom/ogr2osm:latest"
DIGIROAD_SHAPEFILE="${PWD}/data/digiroad_cropped.shp"
docker run \
  --rm \
  --volume "${PWD}/data:/data" \
  "${DOCKER_TAG}" \
  --output "/data/${DIGIROAD_SHAPEFILE}.osm" \
  --force \
  "/data/${DIGIROAD_SHAPEFILE}"
```

A similar processing step could be done by Hootenanny, as well.

### 6. Run hootenanny to conflate Digiroad with the OSM tram network

[Hootenanny](https://github.com/ngageoint/hootenanny) conflates maps.

It has a Dockerfile for the command-line interface (`./docker/hoot_commandline/build_image.sh`).
Build it.

Conflate OSM tram network on top of Digiroad by running Hootenanny with suitable settings.

The suitable settings have not been found yet.

Some script to get started with:
```bash
#!/bin/bash

set -Eeuxo pipefail

IMAGE_NAME=hoot_core
IMAGE_VERSION=latest
TIMESTAMP="$(date --utc '+%Y%m%dT%H%M%S.%NZ')"
CONTAINER_NAME="hoot_${TIMESTAMP}"

docker run --rm -it \
  --name="${CONTAINER_NAME}" \
  --shm-size=4g \
  -v "${HOME}/prog/roelderickx/ogr2osm/results:/home/dev" \
  "${IMAGE_NAME}:${IMAGE_VERSION}" \
  hoot conflate \
    -C UnifyingAlgorithm.conf \
    -C AttributeConflation.conf \
    digiroad_uusimaa_2_k_2021_2_tram_research_area_extract.shp.osm \
    osm_tram_research_area_extract.osm \
    conflated_"${TIMESTAMP}".osm \
   --stats stats_"${TIMESTAMP}".json
```

## Further development

Perhaps Hootenanny would retain more information for conflation if Hootenanny was used for format and coordinate transformation and cropping.
