---
title: P3, Cleaning up OpenStreetMap data
layout: post
---

*Now it is time for the third post dedicated to the Udacity nanodegree and this
time the theme is data wrangling. Rather than focus on the analysis of the data
we will work with checking it's consistency and looking for possible human
errors that might have occured when the data was created. Tools used this time
was my text editor of choice, VIM, together with MongoDB and the accompaning
driver for python called pymongo.*

## Introduction and overview

Now it is time to take a swing at what many sources say is the main part, of
what a professional data scientist do, cleaning and preparing data.

To have something to work with I have downloaded OpenStreetMap data of the
surroundings where I grew up. Just to make sure that we do not get tempted and
start cleaning everything manually I decided to download a chunk of data with a
respectable size of ~75Mb.

Since the input is provided by humans there are bound to be some mistakes or
inconsistencies. The plan is to take a look at city names, street addresses, and
post numbers to see if we can find anything we need to straighten out.

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

There are an abundance of different tags, for example to describe a building you
add 'building=yes' to a closed way and a fast food restaurant would be a node
having a 'amenity=fast_food' tag attached to it. 

For a quick example, see
[this](https://wiki.openstreetmap.org/wiki/OSM_XML#OSM_XML_file_format_notes)
example from the wiki page.  Just to give an example before we start this is a
short extract from the file we are going to use.

### A small note on character encoding

If you are from a part of the world, like me, where not only the standard
characters from A to Z are being used you might get some initial troubles when
trying to handle text strings in your local language. 

  

  
First of all you will need to add the following row to the top line of your
file to tell python that you want to use utf-8 as your character encoding.

```
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

1. Audit - Find problematic elements in the data set
2. Plan cleaning - Identify cause, define cleaning operation, and test it
3. Execute - Execute the plan(usually in the form of a python script)
4. Manually correct - Manually correct any remaining problematic elements

Instead of trying to clean everything all at once we will limit ourselves to
cleaning one aspect per iteration, creating a new script for each new aspect we
decide to clean and then when ready we combined the scripts to create our final 
fully cleaned .json data file.

Now it is finally time to get our hands dirty with some python and we start by 
looking at the data set's address field. The first subfield we look at is the
postal code using the following python function to audit our data.

```
POST_CODE_REGEX = re.compile(r'^\d{3} \d{2}$')

def audit_post_code(unknown_codes, post_code):
    '''
    Function used to check if the provided post_code is conforming to the
    expected name format. If not add the post_code to the provided
    unknown_codes set.
    '''
    if not POST_CODE_REGEX.match(post_code):
        unknown_codes.add(post_code)

    if post_code[:2] >= 42 and post_code[:2] <= 54:
        unknown_codes.add(post_code)

    return
```

What this function does is that it compares all post codes we find in our
data set with the regular expression given and if the code doesn't match the
format we expect it to have we add it to the 'unknown_codes' set. By the way,
the format for Swedish post codes is two set of 3 and 2 numbers separated by a
space, visit [Wikipedia](https://en.wikipedia.org/wiki/Postal_codes_in_Sweden)
to learn more.  

To add a bit more finesse we also check that the two first digits in the postal
code is in the range between 42 and 54 which is the positional numbers
indicating the region our map data is taken from. Anything not starting in this
range is probably wrong and from a totally different part of the country than
what we are expecting.

Running the above code and printing out the postal codes found in the
`unknown_codes` variable we found that some codes where inputted in the data set
without the separating space we are expecting. 

Now to device our plan for how to to fix this issue we use the below function
that takes a five digit number and splits it up into two groups of 3 and 2
digits. (In hindsight doing this the other way around, treating the postal codes
as a 5 digit number would have been a better choice since handling a string for
a number is far from optimal.)

```
def update_post_code(post_code):
    if post_code[3] != ' ':
        post_code = post_code[:3] + ' ' + post_code[3:]

    return post_code
```

In order to test if our plan worked we run the above lines of code on all the
problematic elements we find before running the auditing code one more time.
This time we were lucky and the above few lines of code solved all the
issues we had found. Would this not have been the case we would have
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
  name. Verified possible place names and then pushed any occurrences of these
  names in front of the street name to make the format consistent.

* Also verified that all addresses in the data set had prefixes, suffixes, or
  names-endings normally used in Swedish street names.

To see the code for how I did these checks and revisions please see the
`audit_city_names.py` and `audit_street_names.py` files on this projects
[github page](https://github.com/jepersson/P3-map).

Finishing this it felt like a good timing to move on to creating our .json file.

## Converting our .osm data file to .json

Converting our data from the .osm format to .json we first need to decide on
what model we should use for our data. While some tags can be directly
transferred from one format to another just by bringing over the key value pair
to .json, some tags are nested with sub tags using the following type of notation
`<Tag>:<Sub-tag>`. One example of this is the address tag that looks like the
following in the .osm data format:

```
<tag k="addr:city" v="Vårgårda" />
<tag k="addr:housenumber" v="1A" />
<tag k="addr:street" v="Hallabergsvägen" />
```

We would like to translate this to something a bit more elegant in our .json
data that looks like this:

```
"address": {
    "city": "Vårgårda", 
    "street": "Hallabergsvägen",
    "housenumber": "1A"
}
```

We will do something similar with the meta data values saved in the element by
grouping together values such as "version", "changeset", "timestamp", "user",
and "uid".

Lastly, we also want to convert the positional coordinates from two separate
"lat" and "lon" key value pairs into one single "pos" entry containing a list
with the "lat" and "lon" values.  

The final resulting data model will look something similar to this:

```
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
addresses (together with any other tags containing sub-tags) while writing our
other key value pairs as is into our .json data file. You can reference the file
in the github repository
[here](https://github.com/jepersson/P3-map/blob/master/process_osm.py).

Writing a script were we run all the elements through the above code snippet
gives us a .json data file at ~85Mb (an 13.5% increase in size) with our newly
cleaned map data.

Now we only have two steps left, to import the .json file into a local MongoDB
instance and use the MongoDB query language (through pymongo in our case) to ask
some question about my old home town.

## Importing the .json datafile to MongoDB

This is actually doesn't take that much effort. To import our newly generated
.json data file we only make sure that we are running our MongoDB instance
before inputting the following command in our command line.

```
$ mongoimport --db p3 --collection maps --file openStreetMapData.osm.json
```

With this command we have now loaded our data containing 395053 documents into
our local MongoDB instance.

## Running queries 

So, when we now have got all our data into MongoDB it would be a waste not to
try running a couple of queries to see what the data set contains.

(To see how to run these queries through pymongo yourself take a look
[here](http://api.mongodb.com/python/current/tutorial.html?))

Number of nodes:

```
> db.maps.find({"type": "node"}).count()
361351
```

Number of ways:

```
> db.maps.find({"type": "way"}).count()
33690
```

Unique users:

```
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

From the above queries one that stands out for me is the last one listing the
most common amenities contained in the data set. Coming from what usually is
called the most coffee shop dense town in Sweden I am surprised to not be able
to see something to that effect listed in the above list (last time I looked
this up were a couple of years ago but, there was somewhere between 20-25 coffee
shops, bakeries or similar establishments present at that time).

Let's see if we can find these lost coffee shops somewhere. My first hypothesis
would be to check if there are any data we are missing hidden in the restaurant
or bakery tags.

Let's check how many restaurants and bakeries there are in addition to the
number of coffee shops.

Number of cafes, restaurants, and bakeries:

```
> agg_results = db.maps.aggregate([
      {"$match": {"$or": [
          {"amenity": "cafe"},
          {"amenity": "restaurant"},
          {"shop": "bakery"}
      ]}},
      {"$group": {"_id": {"amenity": "$amenity", "shop": "$shop"},
                  "count": {"$sum": 1}}}
  ])
{u'count': 5, u'_id': {u'shop': u'bakery'}}
{u'count': 15, u'_id': {u'amenity': u'cafe'}}
{u'count': 32, u'_id': {u'amenity': u'restaurant'}}
```

We can see using the above query that there are plenty of restaurants for it to
be room for some mislabeled coffee shops as well. Let's have a look at what type
of values the cuisine tag for these restaurants are.

List up the different cuisines for restaurants in the data set:

```
> agg_results = db.maps.aggregate([
      {"$match": {"amenity": "restaurant"}},
      {"$group": {"_id": "$cuisine", "count": {"$sum": 1}}}
  ])
{u'count': 1, u'_id': u'italian'}
{u'count': 8, u'_id': u'pizza'}
{u'count': 5, u'_id': u'regional'}
{u'count': 1, u'_id': u'thai'}
{u'count': 1, u'_id': u'chinese'}
{u'count': 16, u'_id': None}
```

Other than seeing that Swedish people seem to strongly favor pizza in front of
any other type of food to eat at a restaurant we also see that we have around 16
restaurants that doesn't have a specified cuisine. We will have to dig a bit
deeper and have a look at what other data we have on these restaurants to see if
we might be able pinpoint our missing coffee shops. To list up all the keys we
find we will use the following python function.

```
def find_all_keys(collection):
    def find_keys_in_doc(doc):
        found_keys = []
        for k, v in doc.items():
            found_keys.append(k)
        return found_keys

    all_keys = []
    for doc in collection:
        all_keys += find_keys_in_doc(doc)
    return sorted(set(all_keys))
```

With the above function we using the following to find all restaurants with any
defined cuisine and save the output into a new MongoDB collection we call
subset.

```
> db.maps.aggregate([
      {"$match": {"$and": [
                              {"amenity": "restaurant"},
                              {"cuisine": {"$exists": 0}}
                          ]}},
      {"$out": "subset"}])
> keys = find_all_keys(db.subset.find({}))
_id: 16
 amenity: 16
 building: 2
 created: 16
 id: 16
 leisure: 1
 name: 5
 node_refs: 2
 pos: 14
 smoking: 1
 source: 1
 type: 16
 wheelchair: 1
```

Given the above results we can see that there actually is not much we can do
that does not require considerable effort. Would there had been address tags for
these positions we could had done a cross check with the Swedish version of the
yellow pages perhaps to find additional data but less than half of these
restaurants we found do not carry much more information than the position, meta
data, and the amenity tag. 

In the beginning I was surprised that there were this much data on the region
given that it is a really minor town in a to begin with small country but I
start to believe that the reason is that the resolution of the data is not that
high. Just to verify this let's run a last query, similar to the one above but
on our whole data set and print out the results.

Occurrences of tags for all data points in the map data set:

```
> keys = find_all_keys(db.maps.find({}))
Bing: 4
FIXME: 34
_id: 395053
access: 157
activation: 2
address: 2275
(...)
wikipedia: 13
wires: 4
workrules: 87
```

Unfortunately, there were way to many keys to list all of them here but giving
the list a quick read though it is possible to say that with exception for the
most basic tags there are both duplicates, inconsistent naming (in Swedish and
English) to name a few of the problems. Since not only the data values but also
the keys to a large extent are made by the users I start to feel that we should
not only had taken a look at the values before creating our .json file and
imported to MongoDB but also the keys. 

This would take far too much time to start over at this point so I save this for
a rainy day sometime in the future instead. But, we can say that while it is
important to have clean and consistent data the metadata is equally important
in order to analyze it.

As a last footnote, a possible way to flesh out the missing parts of the data
set would be to use another data source with information complementing the one
existing OpenStreetMap data. One such source would be the Swedish yellow pages,
[eniro.se](http://www.eniro.se), which contains addresses and names to both businesses
and private residents in Sweden.

I believe that as long as we use the data obtained for personal use it should be
okay, but they might would like to have a word with us should our analysis end
up in any commercial application. Another thing to take into account is
restrictions on the number of requests and amount of data we are able to pull
out from their open API. It is not uncomman that there are restrictions to
minimize the risk of misuse and depending on the size of our data set this might
cause trouble for us.

Thank you for holding out until the end of another of my long posts and I hope
to see you next time when we will look closer at some explorative data analysis.
