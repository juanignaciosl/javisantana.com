---
layout: blog3
published: true
---

# vector tiles

Last week we had the n-th GIS drama about [how Mapbox Vector Tiles should be called](https://twitter.com/pwramsey/status/577959264678850561). I'm actually thiking in creating the GIS version of [rubydramas](https://news.ycombinator.com/item?id=4487963) (it's gone, looks like ruby community has moved to node.js, sorry, io.js). This post is not to talk about naming, we all know standards with company names in the description are never used, look at those ESRI shapefiles...

The objetive of vector tiles (whatever format you use) is to move the data closer to the rendering stage, so it's projected, clipped, transformed, extra precision is removed, filtered using naive filters, encoded and finally compressed. You should take a look at this [Michal Migurski's blogpost](http://mike.teczno.com/notes/postgreslessness-mapnik-vectiles.html) or this talk from [Dane Springmeyer in foss4g](https://2015.foss4g-na.org/sites/default/files/slides/foss4g-2015-sf-springmeyer.pdf) to understand what is the rationale behind them.


CartoDB is a platform that runs on top of postgis, it renders tiles fetching them directly from a spatial query so we pay the price for all the overhead of going to the database, fetching the data, preparing it for render and so on. That's why at some point someone though about removing that dynamic part of the equation. But in the other hand we have all the power of postgres and postgis (you know, spatial indices, geometry functions and so on)

So would it possible to generate a easily vector tile from postgres? Of course is, you can do almost everything in postgres right now with an extension but I'm one of those persons that like to do obvious things. It sounds easy, basically we need:

 1. Get the geometry for a given tile with [CDB_XYZ_Extent](https://github.com/CartoDB/cartodb-postgresql/wiki/CDB_XYZ_Extent). PostGIS makes this pretty fast since if you have an index in the column.
 1. Remove extra precision. A tile is usually 256x256 pixels so you don't need 6 decimal precision. Also this makes a lot of points to be in the same pixel so we can remove them. ST_SnapToGrid to the resque here. It's useful to know what is the resolution for the zoom level you are generating so [CDB_XYZ_resolution](https://github.com/CartoDB/cartodb-postgresql/wiki/CDB_XYZ_Resolution) is handy.
 1. We don't need geometry outside the tile, so ST_ClipByBox2d can remove the geometry outside the tile. This function is only present in postgis 2.2 (currently in development), you can use the slower version [ST_Intersection](http://postgis.refractions.net/docs/ST_Intersection.html)
 1. Finally change coordinate system so coordinates are within 0-255 range, ST_Affine makes the algebra thing easy.

 So here it is, given a query that retusn a resultset with cartodb_id and the_geom_webmercator (a geometry column in 3857):

<script src="https://gist.github.com/javisantana/2b12dcb66958ae0680ff.js"></script>

Notice this query does not manage buffer-size, overzooming and so on, that's pretty easy add tho. Also there is a ``res/20`` that needs an extra explanation. If we used half of the pixel for the snapping we'd soon realize that some polygons and lines are removed pretty soon so using that 20 fixes the thing. I have to say that value was calcualted by hand and there are not maths behind it, why spend hours thinking when with a simple binary search you can fix the thing... The geometry is also simplified after snapping (be sure you do after snapping, the simplify algorithm complexity is higher than the snapping)

**does this work?**

let's try with an extract of OSM planet where the geometry is about 350Mb

<script src="https://gist.github.com/javisantana/31d470bc5e858c6d74b5.js"></script>

the CartoDB Vector Tile (did I say I'm pretty good at naming?) is 44Mb (3.8M gzip compressed) so not that bad.

But we still didn't do anything special with the geometry encoding, we are using WKB to store all the things. Remember that WKB uses 16 bytes per coordinate in a geometry. Mapbox vector tiles use varint encoding of delta values in order to make this smaller. I personally don't like [varint](https://developers.google.com/protocol-buffers/docs/encoding#varints) to encode numbers, It's better to leave the compressor do its work and don't try to be smart playing with bits. But ok, in postgis we have a way to delta-varint all the things, it's called [TWKB](https://github.com/TWKB/Specification):

```
copy (select st_astwkbagg(geom, 0, id) from cdb_tile(0, 0, 0, 'select id as cartodb_id, the_geom as the_geom_webmercator from planet')) TO '/tmp/tile2.cvt';
```

The result is a 9.8Mb (1.8M gzip compressed) tile. Much better and took about the same time to encode it. This also works much better with polygon/lines tables than mixed types, specially when there are a lot of points like in this case.

There a lot of things left, for example, when to use clipping, snapping and simplification (sometimes it's better to send every single geometry than cut), coincident points, attribute optimization (I didn't talk about attributes here because with postgres is pretty clear how to do this)

