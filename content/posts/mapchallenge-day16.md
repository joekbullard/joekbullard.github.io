---
title: "Day 16 - Oceania"
date: 2023-11-15T22:31:46Z
draft: false
tags: ['postgis', '30daymapchallenge', 'postgres', 'docker']
---

It is often said that Perth is one of the most isolated cities on the planet, indeed Wikipedia mentions the following:

> With more than two million residents, Perth is one of the most isolated major cities in the world. The nearest city with a population of more than 100,000 is Adelaide, over 2,100 km (1,305 mi) away. Perth is geographically closer to both East Timor (2,800 km or 1,700 mi), and Jakarta, Indonesia (3,000 km or 1,900 mi), than to Sydney (3,300 km or 2,100 mi).

For today's map I thought I would put the claim to the test and visualise the results. Also, today happens to be PostGIS day - so I thought it would be fitting to use PostGIS. I already have a PostGIS server running, but it would be simple enough to get one up and running [using Docker](https://registry.hub.docker.com/r/postgis/postgis/). The following command should work:

```shell
docker run --name mapchallengedb -e POSTGRES_PASSWORD=dbpassword -p 5432:5432  -d postgis/postgis
```

This will create a PostGIS container mapped to port 5432 of your local machine with a user of 'postgres' and password of 'dbpassword'. If you already have services lisenting on port 5432 you may need to change it. To test the connection try the following psql command:

```shell
$ psql -h localhost -p 5432 -U postgres
```

All things being well, this should connect you to the container. 

To plot cities, I'm going to use the cities dataset from [simplemaps](https://simplemaps.com/data/world-cities). I'm going to load this into PostGIS using ogr2ogr - but you could also use qgis or any other method you prefer.

```shell
ogr2ogr -f PostgreSQL PG:"dbname=postgres user=postgres password=mysecretpassword port=5432 host=localhost" /home/joe/Documents/worldcities.xlsx -nln cities
```
Now we have our cities data, we just need to process it generate our geometry column. Given the data is a global dataset, we can use the [geography type](https://postgis.net/documentation/faq/geometry-or-geography/) for this. Witihn the psql shell or using qgis dbmanager you can use the following:

```sql
alter table cities add column geog geography(POINT,4326);
update table cities set geog = ST_GeogFromText('SRID=4326;POINT(' || lng || ' ' || lat || ')');
create index cities_geog_gix ON cities USING GIST (geog);
```

We now have our cities data as geography and built a spatial index, you can load this into qgis and have a look. 

![Cities dataset](/30daymapchallenge2023/day16_cities.png)

You will notice there are a _lot_ of settlements included, so we need to decide on a definition of a city. Given that wikipedia uses a figure of 100,000 we will go with that first.

```sql
select 
a.city as city,
b.city as nearest_neighbor,
round(st_distance(a.geog, b.geog)/1000) as distance
from cities a
cross join lateral
(
	select city, geog
	from cities b
	where a.city <> b.city
	and a.population > 100000
	and b.population > 100000
	order by a.geog <-> b.geog
	limit 1
    ) b
order by distance;
```

This query uses a cross join to combine each row of the table with one another, using the `<-> ` operator ensures the query will be index assisted and thereofre quicker to process. The output returns the nearest neighbour for each city:

![Query results](/30daymapchallenge2023/day16_table.png)

As we can see, Perth is not quite first but a very respectable second. Let's amend the query to include the five nearest neighbours of Perth and plot them.

```sql
select 
a.city as city,
b.city as nearest_neighbor,
round(st_distance(a.geog, b.geog)/1000) as distance,
(ST_Segmentize((ST_MakeLine(a.geog::geometry, b.geog::geometry)::geography),100000)::geometry) as geog
from cities a
cross join lateral
(
	select city, geog
	from cities b
	where a.city <> b.city
	and a.city ='Perth'
	and a.population > 100000
	and b.population > 100000
	order by a.geog <-> b.geog
	limit 5
    ) b
order by distance;
```

And here is the result after a bit of sprucing up in QGIS.

![5 nearest neighbours to Perth](/30daymapchallenge2023/day16.png)