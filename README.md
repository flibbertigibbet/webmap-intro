# Welcome to Web Maps!

We will explore interactive map-making for the web.

Not a coder? Not a problem! We will start out by creating a map on [CartoDB](https://cartodb.com/), which you can do with a free account and without knowing how to code.

Want to see some code? Sure thing! We will get to that, too.

## What's covered:

  - Import a dataset of Philadelphia parks to CartoDB
  - Try out some spatial queries for data analysis:
    - How many parks are within half of a mile of City Hall?
    - What ten parks are closest to City Hall, and how far away are they?
  - Make a web page that loads your CartoDB visualization
  - Make a web page with a [Leaflet](http://leafletjs.com/) map
  - Load a [GeoJSON](http://geojson.org/) file into a Leaflet map

## What's needed:

To follow along with the CartoDB portion, you will want to create a free account. Here is [an example](https://banderkat.cartodb.com/viz/d0d757ec-0ff8-11e6-9c80-0e8c56e2ffdb/public_map) of what a simple map of the parks dataset looks like once it has been imported and shared as a public visualization.

## Fun with CartoDB

We will play with a dataset of Philadelpia park boundaries, which you can download from PASDA [here](ftp://www.pasda.psu.edu/pub/pasda/philacity/data/Philadelphia_PPR_Park_Boundaries201302.zip) (426 kB). Import it into your CartoDB account and create a map from it.

You may want to rename your dataset after it finishes importing to something easier to type. I named mine `philly_parks`.

## Querying PostGIS

CartoDB uses PostGIS geospatial extensions to a PostgreSQL database created for your account, and provides an interface for viewing what's in the database. We're going to try out some PostGIS queries.

Open the sliding pane to the right of the map view. The SQL tab is towards the top. Here we can write custom queries and try them out.

We want to search for parks around City Hall, so first let's draw a circle around City Hall with a half-mile radius. Plug this into the SQL pane and click 'Apply':

```sql
SELECT ST_Buffer(
 ST_Transform(
   ST_SetSRID(
     ST_Point(-75.163431,39.952707),
     4326),
   3857),
  0.5 * 1609) as the_geom_webmercator
```

So what does that do? The query creates a point at City Hall with `ST_Point`, then `ST_Buffer` creates a buffer around it, in meters; that's the radius of the circle drawn. `ST_Transform` and `ST_SetSRID` deal with map projections. For more information on buffering and distances, CartoDB has a tutorial [here](http://academy.cartodb.com/courses/sql-postgis/postgis-in-cartodb/).

A yellow banner pops up at the top of the map; click the link in it to 'create dataset from query'. Now we can have our circle around City Hall saved to easily put on other maps. I named my new dataset `city_hall_half_mile`.

Create a new map with both datasets. (You can also add a layer to an existing map instead). Change the polygon fill color for one of the layers, so they are easier to tell apart. Now, let's filter the parks to only show the ones within two miles of City Hall by putting this query into the SQL pane:

```sql
SELECT philly_parks.* FROM philly_parks,
city_hall_half_mile
WHERE ST_DWithin(
  city_hall_half_mile.the_geom_webmercator,
  philly_parks.the_geom_webmercator,
  1.5 * 1609)
```

`ST_DWithin` filters the second geometry based on its distance in meters from the first geometry.

So which parks lie at least in part within the half-mile circle?

```sql
SELECT philly_parks.* FROM philly_parks,
city_hall_half_mile
WHERE ST_Intersects(
  city_hall_half_mile.the_geom_webmercator,
  philly_parks.the_geom_webmercator)
```

And which parks are fully inside the circle?

```sql
SELECT philly_parks.* FROM philly_parks,
city_hall_half_mile
WHERE ST_Contains(
  city_hall_half_mile.the_geom_webmercator,
  philly_parks.the_geom_webmercator)
```

Okay. What are the ten parks closest to City Hall, and how far away are each?

```sql
SELECT ST_X(ST_Centroid(the_geom)) as longitude,
  ST_Y(ST_Centroid(the_geom)) as latitude,
  park, cartodb_id,
  the_geom_webmercator,
ST_Distance(the_geom::geography,
  ST_PointFromText('POINT(-75.163431 39.952707)',
  4326)::geography) AS distance
FROM philly_parks
ORDER BY the_geom <-> ST_PointFromText(
'POINT(-75.163431 39.952707)', 4326)
LIMIT 10
```

You can see the distances listed by going to the 'Data View', using the button at top. For more information on distance sorting, CartoDB has a tutorial [here](http://docs.cartodb.com/tips-and-tricks/advanced-analysis/#sort-records-by-distance-to-a-point).

## Making a web page that loads a CartoDB visualization

If you click the 'Publish' button in the upper right corner of a map, you get a shareable link, a snippet of code for embedding the map, and a third option that we're going to explore now, which is to load your map using CartoDB.js.

Copy the link in the CartoDB.js section to your map's `viz.json`. We will use that later. If you follow the 'Read more' link above it, it will take you to the CartoDB.js tutorials. Let's start with their source for the 'Getting Started' tutorial, and modify it to load our visualization. Copy the example code from [here](https://raw.githubusercontent.com/CartoDB/cartodb.js/develop/examples/easy.html) into a text editor.

Change the link to the `viz.json` to reference the one for your map. Set the center to be on City Hall, and change the zoom level to zoom in on Philly:

```js
center_lat: 39.952707,
center_lon: -75.163431,
zoom: 15
```

Now if you open the HTML file in a browser, you should see your map. You can see what mine looks like [here](parks_near_city_hall.html).

CartoDB.js uses the Leaflet JavaScript library for displaying web maps. Next we're going to try out Leaflet directly.

## Leaflet


