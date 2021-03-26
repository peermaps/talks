# peermaps: peer to peer maps in the browser

https://github.com/peermaps/talks/blob/main/speakeasyjs-2021-03-26.md

---
# suppose you want to put a map in a web page...

what are your options?

commercial tile hosting

* google maps
* mapbox
* etc

or roll your own tile hosting (hard, annoying)

---
# what is a tile

tiles contain all the features inside a bounding box

* raster tiles - an image file
* vector tiles - binary geojson/svg-ish

different zoom levels, different tiles

rendering engines turn tiles into graphics on the screen

---
# why this sucks

* you need to get tiles from somewhere

"OpenStreetMap data is free for everyone to use. Our tile servers are not."
    - https://wiki.openstreetmap.org/wiki/Tile_usage_policy

* the formats are... unusual (designed around osm/geojson/svg, not around gpus)

---
# tile hosting is very expensive

to make a map on the web, the first step is to type in your payment information

* Google Maps — $7 for each 1,000 map loads, regardless of the map size or zooming/panning by the user ($5.60 with discount for high volume)
* Mapbox — $0.50 for each set of 1000 "map views," which despite the name is not a map view, but a request of 4 or 15 map tiles (depending on map type), rounded up
* Azure Maps — $0.50 for 1000 "transactions," where a transaction is equal to 15 map tiles
* TomTom — $0.50 for 1000 "transactions" ($0.40 with highest volume discounts), where each transaction is equal to 15 map tiles
* HERE — pricing is by bundles, where a standard bundle is $0.50 for 1000 "transactions" (15 tiles)
* MapTiler — $0.05 for additional 1000 map tiles

https://www.programmableweb.com/news/time-to-challenge-google-maps-pricing/elsewhere-web/2018/08/26

---
# tiles lend themselves to centralization

* tiles take a lot of processing to create
* hours, days, or even weeks of compute to build tiles of the whole earth
* you need to have data for the entire area you want to build tiles for
* whenever you update a record, you need to update the tiles the record belongs to

---
# an alternative to tiles: a local database with p2p sync

* query database for all of the features inside of a bounding box
* render those features
* load data from peers sparsely

---
# pulling the rug out from open street map

* if you can make updates and see changes immediately without regenerating tiles
* if you can pull in data from other sources (without regenerating tiles)

* then, you don't need a canonical dataset of the world
* you can have a plurality of datasets, not all of them public necessarily

---
# plurality and decentralization

it doesn't really matter if there is a p2p version of open street map

because open street map is inherently centralized:

* there is one version
* all edits pass through one location: the osm database

---
# osm format

xml: the style at the time

* `<node>`
* `<way>`
* `<relation>`

each feature has an id, but they are in different namespaces

---
# osm format: node example

```
<node id="2430596362"
  lat="31.1876426"
  lon="29.8862615"
  version="4"
  timestamp="2019-11-04T15:57:35Z"
  changeset="76602306"
  uid="9945423"
  user="Eman_Mehrez"
/>
```

---
# osm format: way example

```
<way id="410650621" version="1" timestamp="2016-04-15T19:55:44Z" changeset="38602096" uid="2953895" user="warneke7">
  <nd ref="4124502937"/>
  <nd ref="4124504127"/>
  <nd ref="4124503646"/>
  <nd ref="4124504208"/>
  <nd ref="4124504078"/>
  <tag k="highway" v="residential"/>
  <tag k="source" v="digitalglobe"/>
</way>
```

---
# osm format: relation example

```
<relation id="11078662" version="1" timestamp="2020-05-07T23:03:14Z" changeset="84856412" uid="11136212" user="Baddiboy">
  <member type="way" ref="98236890" role=""/>
  <member type="way" ref="98236890" role=""/>
  <member type="way" ref="801424198" role=""/>
  <member type="way" ref="801424198" role=""/>
  <tag k="type" v="canal"/>
  <tag k="type" v="canal"/>
</relation>
```

---
# osm pbf

layout on disk of a pbf file:

```
[all the nodes]
[all the ways]
[all the relations]
```

no locality: need to process the whole file to find a way ref or relation member record by ID

---
# p2p geo-database goals

* you don't need the whole dataset (there is no whole dataset)
* act on small chunks of data (denormalized)
* easy to write your own rendering stack

---
# peermaps components

* eyros - p2p geo-database (rust, wasm)
* ingest scripts - import planet-osm.pbf into peermaps db (rust)
* georender - display results from database queries as graphical maps (js)

---
# eyros: peermaps database

* builds on bkd tree work for osm-p2p, used in mapeo
* built around sparse replication
* emphasis on batch write performance
* written in rust, compiles to wasm with js wrapper

* npm package: https://www.npmjs.com/package/eyros
* rust crate: https://crates.io/crates/eyros

---
# eyros: design

based on interval and bkd trees:

* bkd trees for very good batch write performance (log structured merges)
* interval trees so you query all features that intersect a bounding box

(features can be points or have bounding boxes of their own)

(so you don't need to do polygon clipping or duplicate features across boundaries as with tiles)

---
# eyros: data format

based on files (easy to distribute with ipfs and hyperdrive)

* db/meta - stores a list of roots and the `next_tree` id
* db/tree/ID - stores a tree file

---
# eyros: tree file format

each tree is a file with a collection of branches:

* branch.pivots - lines to bisect the map, alternating hamburger (y-axis) and hotdog (x-axis)
* branch.intersections - set of nodes that intersect each pivot
* branch.nodes - set of nodes between pivots

each node is:

* branch pointer - offset into the current file of another branch
* data block - inline data and IDs of external tree files

---
# p2p databases: reads vs writes

unlike common setups for relational databases backing a web frontend,

you will have a lot more writes in the read/write ratio, so perf is more important

because when you sync with another peer, you often need to ingest that data into your local db
(to build materialized views with a kappa architecture)

---
# kappa architecture vs sparse clone

* kappa architecture: materialized views built from append-only logs
* sparse clone: read chunks of a p2p archive without downloading the whole thing

working toward a hybrid sparse kappa architecture model,
experiments with peer queries

---
# what we're building toward

* p2p map stack that can scale to planet-osm sized databases
* p2p map editing without going through osm point of centralization
* updating the data updates the map rendering on the client with no extra batch processing / tile cutting

---
# peermaps planet-osm import data flow

```
planet-osm (56.2 GB pbf, 1.385 TB as xml)
-> peermaps ingest: uses georender-pack.rs (rust)
-> produces eyros db
-> web map queries eyros
-> web map unpacks with georender-pack (js)
-> georender-pack produces webgl buffers
-> webgl buffers rendered with mixmap (using regl)
-> extras (label placement)
```

---
# why we chose to make our own map rendering pipeline

* really complicated to make a custom visualization for open street map data
* common osm formats designed around db dump of osm, not for rendering
* opportunity to make map rendering really fast and easy
    * low-single digit # of milliseconds to convert database results into webgl buffers
    * other designs for vector tiles do triangularization and tag processing at runtime on the client
    * moves complicated type heuristics checks into the format (osm-is-area etc)

---
# designed around shaders and webgl, not geojson nor osm xml

* if you want fast graphics, use webgl
* provide data how webgl wants it

---
# georender-pack

* encodes osm data into a compact buffer format
* decodes buffer format data into webGL buffers
* decoded data is consumed by the shaders

---
# buffer format

## FEATURE: AREA

```
| Type [Value]     | Description              |
|------------------|--------------------------|
| U8 [0x03]        | feature type             |
| VARINT           | type                     |
| VARINT           | id                       |
| VARINT           | p_count (# of positions) |
| POSITION[p_count]| positions                |
| VARINT           | c_count (# of cells)     |
| CELL[c_count]    | cells                    |
| LABEL[...]       | labels                   |
```

https://github.com/peermaps/docs/blob/master/bufferschema.md

---
# georender-pack encoding

* triangularization (mapbox, tangram do earcut at runtime)
* osm nodes/ways/relations => points/lines/areas
* osm collection of tags => 1 `featureType` per element (eg `amenity.library`) that determines render style

---
# osm tags => featureType

osm elements can have many tags, not all of which are important for rendering

```
<way id="410650621" version="1" timestamp="2016-04-15T19:55:44Z" changeset="38602096" uid="2953895" user="warneke7">
  <nd ref="4124502937"/>
  <nd ref="4124504127"/>
  <nd ref="4124503646"/>
  <nd ref="4124504208"/>
  <nd ref="4124504078"/>
  <tag k="highway" v="residential"/>
  <tag k="source" v="digitalglobe"/>
</way>
```

---
# featureType priorities

```
{
"landuse.*": {
    "priority": 10
  },
  "man_made.lighthouse": {
    "priority": 70
  },
  "military.*": {
    "priority": 70
  },
  "place.*": {
    "priority": 10
  },
  "power.*": {
    "priority": 100
  },
  "public_transport.*": {
    "priority": 50
  }
}
``` 

---
# features.json

* maps features ("key.value" from tags) to integers for use in rendering

sample values:

```
"amenity.bureau_de_change": 45,
"amenity.bus_station": 46,
"amenity.cafe": 47,
"amenity.car_rental": 48,
```

(osm features: https://wiki.openstreetmap.org/wiki/Map_Features)

---
# stylesheet

* styles are set per feature type (eg "landuse.industrial" or "amenity.*")
* there are default styles for all features, but specifics can be added

sample style:

```
"highway.footway": {
  "line-fill-width[zoom<10]": 1,
  "line-fill-width[16>zoom>=10]": 2,
  "line-fill-width[zoom>=16]": 3,
  "line-fill-color": "#b11000",
  "line-fill-style": "dash",
  "line-stroke-width": 1,
  "line-stroke-color": "#aaaaaa",
  "line-stroke-style": dash,
  "line-stroke-dash-length": "short",
  "line-stroke-dash-gap": 50
},
"highway.motorway": {
  "line-fill-width[zoom<10]": 3,
    // ...
```

demo: https://kitties.neocities.org/mixmap-georender/kharkivdemo.html

---
# stylesheet => png texture

georender-style2png creates a png texture from a stylesheet json file

```
$ georender-style2png -s style.json -o texture.png
```
---
# georender style texture format

```
[point section]
[line section]
[area section]
[sprite section (still working on this)]
```

* each pixel is rgba
* each featureType is a 1 pixel column (x=featureType)
* each featureType may have 1 or more rows (y-axis) to store style data
* each featureType repeats its rows down the y-axis for each zoom level (20 zoom levels)

sample png: https://kitties.neocities.org/mixmap-georender/style.png

---
# glsl-georender-style-texture

glslify module to look up the style data for a particular feature and zoom level from the style texture

(this module is used in shader code)

```
#pragma glslify: Line = require('glsl-georender-style-texture/line.h')
#pragma glslify: readLine = require('glsl-georender-style-texture/line.glsl')

Line line = readLine(texture, featureType, zoom, featureCount);
// line.fillColor is an rgba vec4
// line.strokeColor is an rgba vec4
// line.fillStyle is a float
// line.zindex is a float
// etc...
```

---
# mixmap-georender

* uses mixmap, a webgl map viewer
* uses georender-pack decoder to convert buffer format data => webGL buffers
* reads style texture png
* passes all this data to the shaders

---
# labels

positioning labels above features so they don't overlap with other labels

"Automatic text placement is one of the most difficult, complex, and time-consuming problems in mapmaking and GIS (Geographic Information System)."

https://substack.net/wip/label2.html

---
# project status

working:

* eyros (js and rust)
* georender-pack (js and rust)
* georender-style2png
* mixmap
* mixmap-georender
* label-maker

in progress:

* peermaps ingest with continuous updates
* stylesheets for labels
* sprites
* seeding tools
* text rendering
* hooked up end-to-end in the browser

https://substack.net/eyros-georender-demo/

---
# project status

not even started yet:

* editor
* p2p collaboration
* routing / directions
* mobile
* filtering data for zoomed-out views

---

https://peermaps.org/
https://github.com/peermaps/

---
