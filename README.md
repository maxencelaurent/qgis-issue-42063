# Start the stack

```bash
docker compose up
```

# Test first QGIS server

First server (`http://localhost:8080/ows/`) serves the `qgis_projects_1/test.gqs` project, which contains one single vector layer which is a names `vector_bat`.

Here is a simple `GetMap` query.

```bash
http://localhost:8080/ows/?service=WMS&map=/io/data/test.qgs&request=GetMap&version=1.3.0&width=512&height=512&crs=EPSG:2056&BBOX=2539106,1184111,2539880,1184814&LAYERS=vector_bat
```

The LegendGraphic is fine:
```
http://localhost:8080/ows/?service=WMS&map=/io/data/test.qgs&request=GetLegendGraphic&version=1.3.0&LAYERS=vector_bat
```

# Test 2nd QGIS server
The second QGIS server publishes the  `qgis_projects_2/test_2.gqs` project. It contains one WMS layer, which is the one provided by the first QGIS server.

The `GetMap` is working as expected.

```
http://localhost:8081/ows/?service=WMS&map=/io/data/test_2.qgs&request=GetMap&version=1.3.0&width=512&height=512&crs=EPSG:2056&BBOX=2539106,1184111,2539880,1184814&LAYERS=vector_bat_as_wms
```

The cascaded WMS query is clearly visible in the logs:
```
...
qgis-issue-42063-qgisserver2-1  | 10:39:01 INFO Server[25]: ******************** New request ***************
qgis-issue-42063-qgisserver2-1  | 10:39:01 INFO Server[25]: Request URL: http://localhost:8081/ows/?service=WMS&map=/io/data/test_2.qgs&request=GetMap&version=1.3.0&width=512&height=512&crs=EPSG:2056&BBOX=2539106,1184111,2539880,1184814&LAYERS=vector_bat_as_wms
...
qgis-issue-42063-qgisserver1-1  | 10:39:01 INFO Server[25]: ******************** New request ***************
qgis-issue-42063-qgisserver1-1  | 10:39:01 INFO Server[25]: Request URL: http://172.18.0.1:8080/ows/?MAP=/io/data/test.qgs&SERVICE=WMS&VERSION=1.3.0&REQUEST=GetMap&BBOX=6.640780999999999601%2C46.80535700000000077%2C6.651016000000000261%2C46.81175499999999801&CRS=CRS%3A84&WIDTH=702&HEIGHT=439&LAYERS=vector_bat&STYLES=&FORMAT=image%2Fpng&DPI=90&MAP_RESOLUTION=90&FORMAT_OPTIONS=dpi%3A90&TRANSPARENT=TRUE
...
qgis-issue-42063-qgisserver1-1  | 10:39:01 INFO Server[25]: Request finished in 34 ms
...
qgis-issue-42063-qgisserver2-1  | 10:39:01 INFO Server[25]: Request finished in 391 ms
```

Nonetheless, the GetLegengGraphic query does not work: generated images only contains the layer title.

```
http://localhost:8081/ows/?service=WMS&map=/io/data/test_2.qgs&request=GetLegendGraphic&version=1.3.0&LAYERS=vector_bat_as_wms
```

In the logs, it's seems the `GetLegengGraphic` query is cascaded as expected, but the server is not waiting the result.

```
# [QGIS first query ]
qgis-issue-42063-qgisserver2-1  | 10:47:36 INFO Server[25]: ******************** New request ***************
qgis-issue-42063-qgisserver2-1  | 10:47:36 INFO Server[25]: Request URL: http://localhost:8081/ows/?service=WMS&map=/io/data/test_2.qgs&request=GetLegendGraphic&version=1.3.0&LAYERS=vector_bat_as_wms
...
# [cascaded query starts]
qgis-issue-42063-qgisserver1-1  | 10:47:36 INFO Server[25]: ******************** New request ***************
qgis-issue-42063-qgisserver1-1  | 10:47:36 INFO Server[25]: Request URL: http://172.18.0.1:8080/ows/?MAP=/io/data/test.qgs&SERVICE=WMS&VERSION=1.3.0&REQUEST=GetLegendGraphic&LAYER=vector_bat&FORMAT=image/png&STYLE=default&SLD_VERSION=1.1.0&TRANSPARENT=true
...
# [first query finished before cascaded one]
qgis-issue-42063-qgisserver2-1  | 10:47:36 INFO Server[25]: Request finished in 3 ms
...
# [cascaded query finished]
qgis-issue-42063-qgisserver1-1  | 10:47:36 INFO Server[25]: Request finished in 17 ms

```


I double check that by sniffing http traffic with wireshark.


Looking in the code, it looked to me that :

https://github.com/elpaso/QGIS/blob/7411e9bbb009402cff6eb2a4bb695eab1b5c33f8/src/core/layertree/qgslayertreemodellegendnode.cpp#L1265

is called asynchronously

from somewhere around
https://github.com/qgis/QGIS/blob/master/src/server/services/wms/qgswmsgetlegendgraphics.cpp

https://github.com/qgis/QGIS/blob/master/src/server/services/wms/qgswmsrenderer.cpp


Linked with:
https://github.com/3liz/lizmap-web-client/issues/1571

