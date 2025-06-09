---
title: "TIL how to use Java spatial libraries in Clojure"
date: 2025-06-09T21:39:16+01:00
draft: false
tags: ['clojure', 'spatial']
---

Being able to utilise Java libraries through interop is a really powerful Clojure feature. One such example that I've been using recently is Geotools, a geospatial library - specifically, I have been usign it to create and write to a [GeoPackage.](https://www.geopackage.org/). This post will quickly run through the basics of accessing Java libraries through Clojure - this will be followed by another post showing how to use this library to create a GeoPackage.

I should add at this point - I have absolutely zero experience working with Java, so I suspect what is shown below will be riddled with flaws, but it has worked for me.

If you're familiar with python then you've probably encountered a `requirements.txt` file - in short this lists the dependencies for a project. For example you might have the following:

```txt
# example requirements.txt
pandas>=1.8
```

The Clojure equivalent is `deps.edn`, this file is usually found in the project root. If, like me, you've set your Clojure project up in VSCode using [Calva](http://caliva.io), you will find your `deps.edn` file has the following:

```clojure
{:deps {org.clojure/clojure {:mvn/version "1.12.0"}}}
```

This is the barebones needed for a Clojure project, indicating all that's required is Clojure itself and no further libraries. We are looking to utilise Geotools, an example of an entry you would use is as follows:

```clojure
:deps
{org.geotools/gt-geopkg {:mvn/version "30.1"}}
```

Here we have specified `gt-geopkg` as a dependency. One slight complication with GeoTools is that it publishes to a different repo. By default, the following repos are checked when attemting to install a dependency:

```clojure
{"central" {:url "https://repo1.maven.org/maven2/"}
 "clojars" {:url "https://repo.clojars.org/"}}
```

To access GeoTools, you need to add the following to your `deps.edn`:

```clojure
{:mvn/repos {"OSGeo" {:url "https://repo.osgeo.org/repository/release"}} ;;new
 :deps
 {org.clojure/clojure {:mvn/version "1.12.0"}
  org.geotools/gt-geopkg {:mvn/version "30.1"}
```

This adds the OSGeo repository - if you now end the current REPL session and jack-in again, the dependency will be installed. We can test this out by creating a new Clojure file and pasting the following in:

```clojure
(ns get-started.create-gpkg
  (:import [org.geotools.geopkg GeoPackage]
           [java.io File]))

(defn create-gpkg [filepath]
  (let [file (File. filepath)
        gpkg (GeoPackage. file)]
    (.init gpkg)
    gpkg))

(create-gpkg "output.gpkg")
```

This will create an empty (and currently invalid) GeoPackage called `output.gpkg` in your project root. The next post will cover defining the correct GeoPackage system tables.
