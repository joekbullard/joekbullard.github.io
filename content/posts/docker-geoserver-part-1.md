---
title: "Docker Geoserver Part 1"
date: 2022-11-21T01:22:15Z
draft: false
---

![Docker logo](https://joekbullard-django-blog.s3.amazonaws.com/media/markdownx/bc55f853-76c7-4bcb-b0ea-a2ea590d0f8c.png)

This is the first in a small series that will go through the process of compiling an open source spatial stack to store, serve and visualise spatial data. Given we will be working with a number of containers it is beneficial to work with docker-compose. 

Note: While it is possible to use Postgres in a docker container, this is not recommended for production purposes. A managed service such as AWS RDS is a better option in these instances. 

Pre-requisites: 

*  [Docker](https://docs.docker.com/get-docker/) 
*  [Docker compose](https://docs.docker.com/compose/install/) 
*  An IDE will also make things easier, I use VS Code myself.

The applications themselves will be sorted by Docker images.

This article will focus on setting up and configuring a PostgreSQL database. Because we are working with spatial data, we will be using the [kartoza postgis image](https://hub.docker.com/r/kartoza/postgis/).

First, let’s make a new directory and navigate into it.

```console
$ mkdir open-stack && cd open-stack
```

Now let’s set up a docker-compose file to start the process of building our services.

```console
$ touch docker-compose.yml
```

Open the `docker-compose.yml` and enter the following:

```docker
version: '3.8'

services:
  db:
    image: kartoza/postgis:14-3.1


    volumes:
      - ./data/db:/var/lib/postgresql/data

volumes:
  postgres_data:
```

Let’s break this down line by line, `version` details the Compose file format, note that new versions of docker-compose require more current versions of Docker Engine. 3.8 is the most recent version but it’s possible to use older versions if you’re using an older version of Docker for whatever reason - you will just need to adjust this number accordingly.

Next we define our `services`, at the moment we are only defining one service, `db`. Within this service we set our image to `kartoza/postgis:14-3.1
` - this is the image we will be using, we also configure a `volume` that will contain the data when the container is running.

Now we have a docker-compose file set up, let’s test it out.

```console
$ docker-compose up
```

Now you should run into a few errors here, this is because we haven’t configured any of the required environmental variables. The output from the terminal should give you some indication of what is required:

```shell
db_1  | Error: Database is uninitialized and superuser password is not specified.
db_1  |        You must specify POSTGRES_PASSWORD to a non-empty value for the
db_1  |        superuser. For example, "-e POSTGRES_PASSWORD=password" on "docker run".
db_1  | 
db_1  |        You may also use "POSTGRES_HOST_AUTH_METHOD=trust" to allow all
db_1  |        connections without a password. This is *not* recommended.
db_1  | 
db_1  |        See PostgreSQL documentation about "trust":
db_1  |        https://www.postgresql.org/docs/current/auth-trust.html
```

So it’s clear we need to specify a password, in addition we will also set the superuser name, port and db name. To do that we could include these as raw strings in the docker-compose file, however, it’s considered better practice to use a .env file that contains the variables. So let’s create a .env file

```shell
$ touch .env
```

Open the `.env` file and configure the settings for the database, note you don’t have to use the settings below and are free to adjust accordingly.

```
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
DATABASE_DB=open_stack_db
``` 

Now we need to adjust the docker-compose to include these variables. I have also included `ALLOW_IP_RANGE` as this allows connections from outside of the docker container, for example if you want to view the data within qgis.

```docker
version: '3.8'

services:
  db:
    image: kartoza/postgis:14-3.1
    env_file:
      - ./.env
    environment:
      - POSTGRES_DB=${DATABASE}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASS=${POSTGRES_PASSWORD}
      - ALLOW_IP_RANGE=0.0.0.0/0
    volumes:
      - ./data/db:/var/lib/postgresql/data
    ports:
      - 5432:5432


volumes:
  postgres_data:
```

Here we specify the location of our .env file. Now re-run the docker-compose:

```shell
$ docker-compose down -v
$ docker-compose up -d
```

Note the `-d` flag, this is a shorthand for `--detached ` which means the container runs in the background. The container should now start correctly, you can check this using `docker container ls` - your postgis container should be shown as running.

```shell
CONTAINER ID   IMAGE             COMMAND                  CREATED         STATUS         PORTS      NAMES
bfc0c956e446   postgis/postgis   "docker-entrypoint.s…"   6 minutes ago   Up 4 minutes   5432/tcp   open-stack_db_1
```

We can also access the database container using psql:

```shell
$ docker-compose exec db psql -U postgres -h localhost -d open_stack_db
``` 

Once successful you should see the following:

```psql
open_stack_db=# 
```

Voila! We now have a database up and running. Now let’s get some spatial data into the database. There are a myriad of ways to get data into a database, for the purpose of development it is often beneficial to automate the process of importing at container start up. Fortunately this is quite a straightforward process using Docker. 

First we need some data, this can be pretty much anything. In this example I have used [Bristol parks and green space sites](https://opendata.bristol.gov.uk/explore/dataset/parks-and-greens-spaces/table/?location=13,51.49672,-2.5781&basemap=jawg.streets) from the Bristol open data portal. You can download as a shapefile then convert to a sql dump 
using ogr2ogr, something like this should work:

```shell
$ ogr2ogr --config PG_USE_COPY YES -f PGDump bristol_green_space.sql parks-and-greens-spaces.shp -lco GEOMETRY_NAME=geom -nlt PROMOTE_TO_MULTI
```
Once we have the sql dump of the data, we need to copy it into the project directory, it can either be in the project root or a given folder, it’s up to you. Next, all we have to do is add the following line to docker-compose under `volumes` within the db service:

```docker
version: '3.8'

services:
  db:
    image: kartoza/postgis:14-3.1
    env_file: # new
      - ./.env # new
    environment:
      - POSTGRES_DB=${DATABASE_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASS=${POSTGRES_PASSWORD}
      - ALLOW_IP_RANGE=0.0.0.0/0
    volumes:
      - ./data/db:/var/lib/postgresql/data
      - ./bristol_green_space.sql:/docker-entrypoint-initdb.d/bristol_green_space.sql
    ports:
      - 5432:5432

volumes:
  postgres_data:
```

Basically any .sql file that is copied into `/docker-entrypoint-initdb.d/` will be executed on startup, by providing relative path to our sql dump this will be mounted as a volume at start of the container. We can test this out by reconnecting to the database. First we need to run docker-compose again:

```console
$ docker-compose down -v
$ docker-compose up -d
```

Then access the database:

```shell
docker-compose exec db psql -U postgres -h localhost -d open_stack_db
```
And finally, run the `\dt` meta command:

```psql
open_stack_db=# \dt
                   List of relations
  Schema  |          Name           | Type  |  Owner   
----------+-------------------------+-------+----------
 public   | parks_and_greens_spaces | table | postgres
 public   | spatial_ref_sys         | table | postgres
 topology | layer                   | table | postgres
 topology | topology                | table | postgres
```
Great, we can see that the import was successful, that's the end of this part of the process, next time we will be tackling GeoServer.