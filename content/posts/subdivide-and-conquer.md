---
title: "Subdivide and Conquer"
date: 2022-11-21T01:12:57Z
draft: false
---

Dealing with large and complex geometries in postgis can be computationally expensive, even with a spatial index. To understand why this is we need to have some understanding of how a spatial index works. In simple terms, a spatial index is a bounding box that encapsulates the extent of a geometry, using a spatial index can rapidly speed up spatial processing by first checking which bounding boxes intersect - a comparatively quick process - as opposed to which raw geometries intersect - potentially a lot slower - and more so as the size of your dataset increases. If you want to learn a little then the [postgis documentation](https://postgis.net/workshops/postgis-intro/indexing.html) does a good job of explaining it further.

So with this in mind, let's get back to the large and complex geometries. When you're looking to identify spatial relationships between geometries that cover geographically large areas comprising a vast number of vertices - think a polygon of a large country such as Russia or USA - you end up with vast bounding boxes. An example I've been grappling with recently involved intersecting large local authorities in Scotland with 100s of 1000s of rows of land cover and soil classifcation types. In these cases, even with a spatial index improving efficiency, you're still looking at undertaking a vast number of spatial operations - taking a long time to complete.

But what if there was a way around this? Knowing that large complex geometries = slow, we can assume that the opposite is true of small and simple geometries. It is possible to apply this logic by using the `ST_Subdivide` function. In effect, this function breaks a polygon down into smaller parts, as these smaller parts cover a smaller geographic extent, the bounding box covers a much smaller area and will therefore intersect with far fewer geometries - speeding up processing time.

Let's take an example using the district borough unitary layer included with the [OS Boundary-Line](https://www.ordnancesurvey.co.uk/business-government/products/boundaryline) dataset.

```sql
SELECT geom
FROM district_borough_unitary
WHERE name ILIKE 'highland'
```

![Highland standard](/highland.jpg)

Now let's try adding a bounding box to that geometry to see how far it extends.

```sql
SELECT ST_Envelope(geom)
FROM district_borough_unitary
WHERE name ILIKE 'highland'
```

![Highland bounding box](/highlandbbox.jpg)

As you can see, the bounding box that forms the index covers a vast area beyond the extent of the original geometry, this means that when spatial queries are running against this geometry, all those areas will be included, dramtatically increasing processing time.

Let's do the same but using `ST_Subdivide` on the geometry.

```sql
SELECT ST_Subdivide(geom)
FROM district_borough_unitary
WHERE name ILIKE 'highland'
```

![Highland subdivide](/highland_subdivide.jpg)

As you can see, the original geometry has been broken down into a large number of smaller polygons, each of which has a much smaller bounding box. So what impact does this have on query performance? Let's investigate with a spatial query against the [HABMOS dataset](https://gateway.snh.gov.uk/natural-spaces/dataset.jsp?dsid=HABMOS) to find out how many forest polygons intersect our study area.

First, let's try with the plain geometry:

```sql
SELECT COUNT(*)
FROM 
	public.district_borough_unitary,
	public.habmos
WHERE 
	ST_Intersects(geom, wkb_geometry)
	AND habitat ILIKE '%forest%'
```
 
So I went and made a cup of tea, came back and aborted the query as it was taking so long (this was carried out on postgres running on docker environment on my raspberry pi, so not the beefiest of hardware, but still)
 
And using the the subdivded geometry....
 

```sql
SELECT COUNT(*)
FROM  
	public.subdivded_geom,
	public.habmos
WHERE 
	ST_Intersects(geom, habmos.wkb_geometry)
	AND habitat ILIKE '%forest%'
```

This completed in just over 7 seconds, or 7 seconds and 251ms to be precise. So a real stark difference between the two, demonstrating clearly why it can be of great benefit to `ST_Subdivide` to conquer those larger polygons!