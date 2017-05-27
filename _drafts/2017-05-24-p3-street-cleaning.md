---
title: P3, Cleaning up OpenStreetMap data
layout: post
---

*This has not been written yet!!!*

## Introduction and overview

Now it is time to take a swing at what many sources say is the main part,
time wise, of what a professional data scientist do, cleaning and preparing data.

To have something to work with I have downloaded some OpenStreetMap data of the
surroundings where I grew up. Just to make sure that we do not get tempted and
start cleaning everything by hand I decided to download a .osm file, which is a
type of xml format, of a respectable size (75MB).

Since the input is provided by humans there are bound to be some mistakes in
the data somewhere. I plan to take a look at street addresses and post numbers to
see if we can find any errors we are able to straighten out.

As a last step we convert our cleaned .osm data in a JSON format so that we then
can import in into a local instance of MongoDB. Once we done that we can start
playing around with the data and answer some questions both about user
participation in the data gathering process and about the area itself.

But first...

## What is OpenStreetMap?

Before we start working with the data I believe a small introduction to the
OpenStreetMap(OSM) project is in its place. The following is a short summary
from the [OSM wiki](http://wiki.openstreetmap.org/wiki/Beginners_Guide_1.3).

The data consists of five different types of elements and each element can have
tags attached to it describing what that element is. The five element types are:

* **Node:** Nodes are dots used to that mark locations. Nodes can be separate or
  can be connected.  
* **Way:** Ways are a connected line of nodes. It is used to create roads,
  paths, rivers, and so on.  
* **Closed way:** Closed ways are ways that form a closed loop. Usually forms an
  area.  
* **Area:** Areas are closed ways which are also filled. An area is usually
  implicitly implied when making a closed way.  
* **Relation:** Relation can be used to create more complex shapes, or to
  represent elements that are related but not physically connected. We won't get
  into this now. See the "advanced topics" section for more info.  
*Descriptions taken from the [OSM
wiki](http://wiki.openstreetmap.org/wiki/Beginners_Guide_1.3).*

These are the basic elements that defines the structure of the data set we are
going to work with. Further, each of these elements can carry tags which are
key and value pairs describing what the elements is and it's characteristics.

There are an abundance of different tags for example to describe a building you
add 'building=yes' to a closed way and a fast food restaurant would be a node
having a 'amenity=fast_food' tag attached to it. 

Just to give an example before we start this is a short extract from the file we
are going to use.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<osm version="0.6" generator="Overpass API">
<note>The data included in this document is from www.openstreetmap.org. The data is made available under ODbL.</note>
<meta osm_base="2017-05-24T13:05:34Z"/>
  <bounds minlat="57.8798000" minlon="12.4146000" maxlat="58.1055000" maxlon="13.1342000"/>
    (...)
  <node id="20854524" lat="58.0571727" lon="12.8186978" version="5" timestamp="2017-03-22T19:38:16Z" changeset="47077002" uid="438299" user="Essin">
    <tag k="bus" v="yes"/>
    <tag k="highway" v="bus_stop"/>
    <tag k="public_transport" v="stop_position"/>
  </node>
    (...)
  <way id="492794593" version="1" timestamp="2017-05-11T14:09:02Z" changeset="48593646" uid="5452069" user="Kudso">
    <nd ref="4847719722"/>
    <nd ref="4847719723"/>
    <nd ref="4847719724"/>
    <nd ref="4847719725"/>
    <nd ref="4847719722"/>
    <tag k="amenity" v="recycling"/>
    <tag k="landuse" v="landfill"/>
    <tag k="recycling_type" v="centre"/>
  </way>
    (...) 
  <relation id="7269524" version="1" timestamp="2017-05-22T12:27:10Z" changeset="48887118" uid="1069176" user="landfahrer">
    <member type="way" ref="431403706" role="outer"/>
    <member type="way" ref="436018989" role="inner"/>
    <member type="way" ref="431783552" role="inner"/>
    <member type="way" ref="431783551" role="inner"/>
    <tag k="landuse" v="farmland"/>
    <tag k="type" v="multipolygon"/>
  </relation>
    (...)
</osm>
```

### A small note on character encoding

If you are from a part of the world, like me, where not only the standard
characters from A to Z are being used you might get some initial troubles when
trying to handle these characters. 

First of all you will need to add the following row to the top line of your
file in order to tell python that you want to use utf-8 as your character
encoding..

```python
# -*- coding: utf-8 -*-
```

Also, for some reason print statements don't print out whole sets or arrays properly
but you can solve this with a loop and a print statement printing out each
element separately.

And lastly, for strings in your python script will have to explicitly state them
as unicode string by prepending a u in front of the first quote like this `u'My
unicode string.'`.

I believe this should have been fixed in python 3 but since the nanodegree's
material still is in python 2.7 I am sticking to that version for now.

## Cleaning up the data


