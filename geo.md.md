
```bash
# osm.pbf转gpkg
ogr2ogr -f GPKG pois.gpkg d:\.work\data\geojson\china_pois.osm.pbf --config OSM_CONFIG_FILE "D:\local\QGIS 4.2.2\share\gdal\osmconf.ini"
```