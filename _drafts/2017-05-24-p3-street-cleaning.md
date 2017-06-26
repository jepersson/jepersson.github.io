---
title: P3, Cleaning up OpenStreetMap data
layout: post
---

## Introduction and overview

Now it is time to take a swing at what many sources say is the main part, of
what a professional data scientist do, cleaning and preparing data.

To have something to work with I have downloaded some OpenStreetMap data of the
surroundings where I grew up. Just to make sure that we do not get tempted and
start cleaning everything by hand I decided to download a chunk of data with a
respectable size of ~75Mb.

Since the input is provided by humans there are bound to be some input mistakes.
I plan to take a look at city names, street addresses, and post numbers to see
if we can find any errors we need to straighten out.

After cleaning our data we convert it from the original .osm format to .json in
order to easily import it into a local MongoDB instance. Once we have loaded the
data into our database we try some queries to see if there is anything
interesting to discover about our data set.

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

These are the basic elements that defines the structure of the data set we are
going to work with. Further, each of these elements can carry tags in the form
of key and value pairs describing the parent element and it's characteristics.

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
trying to handle text strings in your local language. 

First of all you will need to add the following row to the top line of your
file to tell python that you want to use utf-8 as your character encoding.

```python
# -*- coding: utf-8 -*-
```

Also, for some reason, print statements don't print out sets or arrays with
strings containing utf-8 characters properly but you can solve this with a loop
and a print statement printing out each element separately.

And lastly, for strings defined in your python code will have to explicitly be
stated as unicode strings by prepending a u in front of the first quote like
this `u'My unicode string.'`.

I believe this should have been fixed in python 3 but since the nanodegree's
material still is targeting python 2.7 I am sticking to that version for now.

## Cleaning up the data

So lets start by planning how we should clean our data set. We will follow an
iterative process given in the class for Data Wrangling from the Data Analyst
Nanodegree. The steps of this process is as described below:

1. Audit - Find problematic element in the data set
2. Plan cleaning - Identify cause, define cleaning operation, and test it
3. Execute - Execute the plan(usually in the form of a python script)
4. Manually correct - Manually correct any remaining problematic elements

Since trying to clean everything all at once we will limit ourselves to cleaning
one aspect of the per iteration, creating a new script for each new aspect we
decide to clean and then when ready we combined the scripts to create our final
cleaned .json data file.

Now it is finally time to get our hands dirty with some python and for this
post we will narrow it down to looking at the data set's address field. The
first subfield we look at is the postal code using the following python
function to audit our data.

```python
POST_CODE_REGEX = re.compile(r'^\d{3} \d{2}$')

def audit_post_code(unknown_codes, post_code):
    '''
    Function used check if the provided post_code is conforming to the
    in the expected name format. If not add the post_code to the
    provided unknown_codes set.
    '''
    if not POST_CODE_REGEX.match(post_code):
        unknown_codes.add(post_code)

    if post_code[:2] >= 42 and post_code[:2] <= 54:
        unknown_codes.add(post_code)

    return
```

What this function does is that it compares all post codes we find in our
data set with the regular expression given and if the code doesn't match the
format we expect it is added to the 'unknown_codes' set. By the way, the format
for Swedish post codes is two set of 3 and 2 numbers separated by a space, visit
[Wikipedia](https://en.wikipedia.org/wiki/Postal_codes_in_Sweden) to learn more.

To add a bit more finesse we do one last check where we verify that the two
first digits in the postal code is in the range between 42 and 54 which is the
positional numbers indicating the region our map data is taken from. Anything
not starting in this range is probably wrong and from a totally different part
of the country than what we are expecting.

Running the above code and printing out the postal codes found in the
`unknown_codes` variable we found that some codes where inputted in the data set
without the separating space we are expecting. 

Now to device our plan how to to fix this we use the below function that takes
a five digit number and splits it up into two groups of 3 and 2 digits. (In
hindsight doing this the other way around, treating the postal codes as a 5
digit number would have been a better choice since handling a string for a
number is far from optimal.)

```python
def update_post_code(post_code):
    if post_code[3] != ' ':
        post_code = post_code[:3] + ' ' + post_code[3:]

    return post_code
```

In order to test if our plan worked we run the above lines of code on all the
problematic elements we find before running the auditing code one more time.
This time we were lucky and the above few lines of code solved all the
problematic elements we found. Would this not have been the case we would have
needed to print out the given `unknown_codes` one more time to see if the
remaining elements could be cleaned programmatically in any other way or if we
need to go in and fix the remaining parts manually.

This is repeated until we are confident that the chosen aspect of our data is
clean enough. Once we come this far we continue by choosing another aspect of
the data set and once again go through the whole above process starting
from the first step.

This time I repeated the process for the city names and street names attributes
and fixed the following errors before deciding that it was enough: 

* One city name containing the postal code instead of the correct data. Since
  there was only one occurrence of this error it felt reasonable to just correct
  it by hand.

* Street names had inconsequent capitalization, used `.title()` method to fix
  this.

* Street names had inconsequent ordering of place names as part of the address,
  sometimes the place name were before and sometimes after the actual street
  name. Verified possible place names and then pushed any occurencies of these
  names in front of the street name to make the format consistent.

* Also verified that all addresses in the data set had prefixes, suffixes, or
  names-endings normally used in Swedish street names.

To see the code for how I did these checks and revisions please see the
`audit_city_names.py` and `audit_street_names.py` files on this projects
[github page](https://github.com/jepersson/P3-map).

Finshing this it felt like a good timing to move on to creating our JSON file.

## Converting our OSM data file to JSON

Converting our data from the .osm format to .json we first need to decide on
what model we should use for our data. While some tags can be directly
transferred from one format to another just by bringing over the key value pair
to .json, some tags are nested with sub tags using the following type of notation
`<Tag>:<Sub-tag>`. One example of this is the address tag that looks like the
following in the .osm data format:

```xml
<tag k="addr:city" v="Vårgårda" />
<tag k="addr:housenumber" v="1A" />
<tag k="addr:street" v="Hallabergsvägen" />
```

We would like to translate this to something a bit more elegant in our .json
data that looks something like this:

```json
"address": {
    "city": "Vårgårda", 
    "street": "Hallabergsvägen",
    "housenumber": "1A"
}
```

We will do something similar with the meta data values saved in the element by
grouping together values such as "version", "changeset", "timestamp", "user",
and "uid".

Lastly,we also want to convert the positional coordinates from two separate "lat" and
"lon" key value pairs into one single "pos" entry containing a list with the
"lat" and "lon" values. 

The final resulting data model will look something similar to this:

```json
{
    "id": "2406124091",
    "type: "node",
    "visible":"true",
    "created": {
        "version":"2",
        "changeset":"17206049",
        "timestamp":"2013-08-03T16:43:42Z",
        "user":"linuxUser16",
        "uid":"1219059"
    },
    "pos": [41.9757030, -87.6921867],
    "address": {
        "housenumber": "5157",
        "postcode": "60625",
        "street": "North Lincoln Ave"
    },
    "amenity": "restaurant",
    "cuisine": "mexican",
    "name": "La Cabana De Don Luis",
    "phone": "1 (773)-271-5176"
}
```

To transform the current .osm data to .json we use the below
`shape_element`-function which groups metadata, positional coordinates and
adresses (together with any other tags containing sub-tags) while writing our
other key value pairs as is into our .json data file.

```python
def shape_element(element):
    node = {}

    if element.tag == "way":
        node['node_refs'] = []

    if element.tag == "node" or element.tag == "way":
        node['type'] = element.tag
        attrs = element.attrib
        node['created'] = {}

        for attr in attrs:
            if attr == "lat" or attr == "lon":
                if "pos" not in node:
                    node['pos'] = []
                if attr == "lat":
                    node['pos'].insert(0, float(element.attrib[attr]))
                elif attr == "lon":
                    node['pos'].insert(0, float(element.attrib[attr]))
            elif attr in CREATED:
                node['created'][attr] = element.attrib[attr]
            else:
                node[attr] = element.attrib[attr]

        for subtag in element.iter('tag'):
            key, value = subtag.attrib['k'], subtag.attrib['v']
            if problemchars.match(key):
                continue
            elif lower_colon.match(key):
                subtag_keys = key.split(":")
                if subtag_keys[0] == "addr":
                    if "address" not in node:
                        node['address'] = {}
                    if subtag_keys[1] == "street":
                        node['address'][subtag_keys[1]] = \
                            audit_street_names.update_street_name(value)
                    elif subtag_keys[1] == "postcode":
                        node['address'][subtag_keys[1]] = \
                            audit_post_codes.update_post_code(value)
                    elif subtag_keys[1] == "city":
                        node['address'][subtag_keys[1]] = \
                            audit_city_names.update_city_name(value)
                    else:
                        node['address'][subtag_keys[1]] = value
                else:
                    node[subtag_keys[1]] = value
            else:
                if ":" not in key:
                    node[key] = value
            if element.tag == "way":
                for subtag in element.iter('nd'):
                    node['node_refs'].append(subtag.attrib['ref'])

        return node

    else:
        return None

```

Writing a script were we run all the elements through the above code snippet
gives us a .json data file at ~85Mb (an 13.5% increase in size) with our newly
cleaned map data.

Now we only have two steps left, to import the .json file into a local MongoDB
instance and use the MongoDB query language to ask some question about my old
home town.

## Importing the .json datafile to MongoDB

This is actually not that much to talk about. To import our newly generated
.json data file we only make sure that we are running our MongoDB instance
before inputting the following command in our command line.

```shell
(p3-map) ~/Projects/P3-map (master) $ mongoimport --db p3 --collection maps --file openStreetMapData.osm.json

2017-06-26T21:17:37.171+0900    connected to: localhost
2017-06-26T21:17:40.158+0900    [#######.................] p3.maps      27.7MB/86.0MB (32.2%)
2017-06-26T21:17:43.158+0900    [###############.........] p3.maps      56.4MB/86.0MB (65.5%)
2017-06-26T21:17:45.922+0900    [########################] p3.maps      86.0MB/86.0MB (100.0%)
2017-06-26T21:17:45.922+0900    imported 395053 documents
```

With this command we have now loaded our data containing 395053 documents into
our local MongoDB instance.

## Asking questions

Number of documents:
```
> collection.find().count()
395053
```

Number of nodes:
```
> db.maps.find({"type": "node"}).count()
361351
```

Number of ways:
```
> db.maps.find({"type": "way"}).count()
33690
Unique users:
> len(db.maps.distinct("created.user"))
184
```

Top #10 contributors:
```
> db.maps.aggregate([
    {"$group":{"_id": "$created.user", "count": {"$sum": 1}}},
    {"$sort":{ "count": -1}},
    {"$limit": 10}
])
{u'count': 243983, u'_id': u'Agatefilm'}
{u'count': 45478, u'_id': u'leojth'}
{u'count': 15708, u'_id': u'tothod'}
{u'count': 11874, u'_id': u'bengibollen'}
{u'count': 10988, u'_id': u'johnrobot'}
{u'count': 10042, u'_id': u'KLARSK'}
{u'count': 8383, u'_id': u'clobar'}
{u'count': 8103, u'_id': u'Gujo'}
{u'count': 4627, u'_id': u'DavidSamuelsson'}
{u'count': 3186, u'_id': u'Cohan'}
```

Most common amenity:
```
> db.maps.aggregate([
    {"$match": {"amenity": {"$exists": 1}}},
    {"$group": {"_id": "$amenity", "count": {"$sum": 1}}},
    {"$sort": {"count": -1}},
    {"$limit": 10}
])
{u'count': 329, u'_id': u'parking'}
{u'count': 55, u'_id': u'place_of_worship'}
{u'count': 44, u'_id': u'bicycle_parking'}
{u'count': 33, u'_id': u'shelter'}
{u'count': 32, u'_id': u'restaurant'}
{u'count': 30, u'_id': u'school'}
{u'count': 27, u'_id': u'grave_yard'}
{u'count': 22, u'_id': u'fuel'}
{u'count': 21, u'_id': u'post_box'}
{u'count': 18, u'_id': u'recycling'}
```


