# Transnet
The Transnet project consists of a set of Python and Matlab scripts for the automatic inference of high voltage power (transmission) grids based on crowdsourced OpenStreetMap (OSM) data. Transnet yields two different models, a Common Information Model (CIM) model and a Matlab Simulink model. The latter can be used to perform load flow analysis. This manual guides you to the Transnet setup and gives several usage examples.

## Setup Transnet Project:
Transnet requires Python-2.7, which can be installed as follows:
```
sudo apt-get install python-2.7
```
Moreover, QGIS has to be installed - the installation guide for Linux systems can be found here:
http://qgis.org/en/site/forusers/alldownloads.html#linux

A few additional Python packages have to be installed:
```
sudo apt-get install python-psycopg2 python-shapely
easy_install PyCIM
```
Finally checkout the Transnet project:
```
git clone https://github.com/OpenGridMap/transnet transnet
```

## Data Preparation
Download OSM data (.pbf) from https://download.geofabrik.de/ for the considered region, e.g. for Europe or Germany. Also download the corresponding shape file (.poly).

Install the LATEST osmosis tool, which is capable of filtering OSM data by (power) tags:
```
mkdir osmosis && cd osmosis
wget http://bretth.dev.openstreetmap.org/osmosis-build/osmosis-latest.tgz
tar xvfz osmosis-latest.tgz
chmod +x bin/osmosis
```
Filter all nodes, ways and relations tagged with 'power=*':
```
osmosis \
  --read-pbf file=’Downloads/germany-latest.osm.pbf’ \
  --tag-filter accept-relations power=* \
  --tag-filter accept-ways power=* \
  --tag-filter accept-nodes power=* \
  --used-node --buffer \
  --bounding-polygon file=’Downloads/germany.poly’ \
  completeRelations=yes \
  --write-pbf file=’Downloads/power_extract1.pbf’
```
Filter all relations tagged with 'route=power':
```
osmosis \
  --read-pbf file=’Downloads/germany-latest.osm.pbf’ \
  --tag-filter accept-relations route=power \
  --used-way \
	--used-node \
  --buffer \
  --bounding-polygon file=’Downloads/germany.poly’ \
  completeRelations=yes completeWays=yes \
  --write-pbf file=’Downloads/power_extract2.pbf’
```
Get all power ways and its nodes (even if they are not power-tagged):
```
osmosis \
  --read-pbf file=’Downloads/germany-latest.osm.pbf’ \
  --tag-filter accept-ways power=* \
  --used-node --buffer \
  --bounding-polygon file=’Downloads/germany.poly’ \
  completeWays=yes \
  --write-pbf file=’Downloads/power_extract3.pbf’
```
Merge first 2 extracts:
```
osmosis \
--read-pbf file=’Downloads/power_extract1.pbf’\
--read-pbf file=’Downloads/power_extract2.pbf’\
--merge \
--write-pbf file=’Downloads/power_extract12.pbf’
```
Merge with 3rd extract:
```
osmosis \
--read-pbf file=’Downloads/power_extract12.pbf’\
--read-pbf file=’Downloads/power_extract3.pbf’\
--merge \
--write-pbf file=’Downloads/power_extract.pbf’
```

## PostgreSQL + PostGIS setup
Transnet relies on a local PostgreSQL + PostGIS installation, which is the host of power-relevant OSM data.
To install PostgreSQL + PostGIS open a terminal and execute the following command:
```
sudo apt-get install postgresql postgis
sudo -u postgres psql -c '\password'
```
Create a PostGIS-enabled database template:
```
sudo -u postgres createdb -U postgres -h localhost transnet_template
sudo -u postgres psql -d transnet_template -U postgres -h localhost -f /usr/share/postgresql/9.1/contrib/postgis-1.5/postgis.sql
sudo -u postgres psql -d transnet_template -U postgres -h localhost -f /usr/share/postgresql/9.1/contrib/postgis-1.5/spatial_ref_sys.sql
sudo -u postgres psql -d transnet_template -U postgres -q -h localhost -c "CREATE EXTENSION hstore;"
```
Create a database using the template:
```
sudo -u postgres psql -U postgres -d transnet_template -h localhost -c "CREATE DATABASE power_de WITH TEMPLATE = transnet_template;"
```
Install osm2pgsql tool for OSM data import:
```
sudo apt-get install osm2psql
```
Import data extract from above into database:
```
osm2pgsql -r pbf --username='postgres' -d power_de -k -s -C 6000 -v --host='localhost' --port='5432' --password --style transnet/util/power.style Downloads/power_extract.pbf
```

