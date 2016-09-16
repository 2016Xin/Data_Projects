# P3. OpenStreetMap Project -- Data Wrangling with MongoDB

### Map Area
**Brooklyn, New York, United States**
- Boundary of Kings County: https://www.openstreetmap.org/relation/369518#map=11/40.6444/-73.9449
- Metro extracts: https://mapzen.com/data/metro-extracts/metro/brooklyn_new-york/
- A smaller sample ([sample.osm](link)) of the map ([brooklyn_new-york.osm](link)) was generated by the [code](link) in project note.

### Exploring and Auditing the Sample DataSet
1. Explore the tags in this file. Different tags and counts:
```python
{'member': 1392,
 'nd': 349542,
 'node': 248527,
 'osm': 1,
 'relation': 174,
 'tag': 282389,
 'way': 49052}
 ```

2. Explore the 'k' value of the all the tags in 'way' element and count each of them.
    - Some interesting 'k' values I would like to explore further:
 ```python
 {'addr:postcode': 33105, 'addr:street': 33135}
 {'FIXME': 4, 'fixme': 5, ...}
 ```
    - Besides, I would like to explore 'k' values start with "Tiger:" as well.

3. Auditing street names and zip code.
    - Surprisingly, almost all of the street names are good. There is only one over­-abbreviated street names: 'St.' ==> {'9th St.''}

    - The format of the postal codes are consistent. However, a lot of them postal codes are not in brooklyn, but in Jersey City, Lower Manhattan or Queens. [NYC zip code](https://www.health.ny.gov/statistics/cancer/registry/appendix/neighborhoods.htm). All the brooklyn postal codes should start with "112**".

4. Auditing tags in 'way' element which are pulled from Tiger GPS data.
    - The 'tiger:name_type' values are all abbreviation. And some of them include multiple names, eg 'Ave; St; Ave'.
```python
'tiger:name_type': {'Ave',
                    'Ave:Pky',
                    'Ave; St; Ave',
                    'Ave;St;Ave',
                    'Blvd',....},
```
    - A lot of the 'tiger:zip_left' equals to 'tiger:zip_right', and 'tiger:zip_left_1' equals to 'tiger:zip_left'. It looks redundant to me. For example:
```python
<tag k="name" v="Harrison Street"/>
<tag k="highway" v="residential"/>
<tag k="tiger:cfcc" v="A41"/>
<tag k="tiger:county" v="New York, NY"/>
<tag k="tiger:zip_left" v="10013"/>
<tag k="tiger:name_base" v="Harrison"/>
<tag k="tiger:name_type" v="St"/>
<tag k="tiger:zip_right" v="10013"/>      
```   
Another example:
```python
<tag k="name" v="112th Street"/>
<tag k="oneway" v="yes"/>
<tag k="highway" v="residential"/>
<tag k="tiger:cfcc" v="A41"/>
<tag k="tiger:county" v="Queens, NY"/>
<tag k="tiger:reviewed" v="no"/>
<tag k="tiger:zip_left" v="11419"/>
<tag k="tiger:name_base" v="112th"/>
<tag k="tiger:name_type" v="St"/>
<tag k="tiger:zip_right" v="11419"/>
<tag k="tiger:zip_left_1" v="11419"/>
<tag k="tiger:zip_right_1" v="11419"/>      
```   
    - Inconsistent format of 'Tiger: reviewed'. According the the [TIGER fixup](http://wiki.openstreetmap.org/wiki/Talk:TIGER_fixup), when it has been reviewed, the "tiger:reviewed" tag is supposed to be removed. So the only value for this tag should be "no". However, the actually value which have been printed out include: 'yes', 'no; yes; no', 'yes; no' and '; no; no'. It is quite confusing.
    ```python
    'tiger:reviewed': {'no', 'no; yes; no', 'yes; no', 'yes', '; no; no'}
    ```

### Problems Encountered in this Sample
1. Over-abbreviated street names and tiger: name_type. They were all updated when converting from XML into JSON format for importing into MongoDB. [PYTHON code](code link)

2. Wrong regions (regions which do not belong to brooklyn) revealed by the wrong zip code. The right zip code for brooklyn is between 11201 - 11256. It can be further explored in MongoDB.

3. Inconsistent format of DATA pulled from tiger GPS.

### Data Overview
This section contains basic statistics about the dataset and the MongoDB queries used to gather them.

##### File sizes
    - brooklyn.osm ......... 666.2 MB
    - brooklyn.osm.json .... 725.4 MB

1. Number of documents
```python
> db.brooklyn.find().count()
> 2975785
```
2. Number of nodes and ways
```python
> db.brooklyn.find({"type":"node"}).count()
> 2485112
> db.brooklyn.find({"type":"way"}).count()
> 490509
```
3. Top 10 contributing user
```python
> db.brooklyn.aggregate([{"$group":{"_id": "$created.user", "count":{"$sum":1}}}, {"$sort": {"count":-1}}, {"$limit":10}])
```

|  ID                                 |  COUNT              |
| ------------------------------------|---------------------|
|  "_id" : "Rub21_nycbuildings"       |  "count" : 1740323  |
|  "_id" : "ingalls_nycbuildings",    |  "count" : 373554   |
|  "_id" : "ediyes_nycbuildings",     |  "count" : 189694   |
|  "_id" : "celosia_nycbuildings",    |  "count" : 117361   |
|  "_id" : "ingalls",                 |  "count" : 105358   |
|  "_id" : "lxbarth_nycbuildings",    |  "count" : 79851    |
|  "_id" : "aaron_nycbuildings",      |  "count" : 42023    |
|  "_id" : "ewedistrict_nycbuildings",|  "count" : 35002    |
|  "_id" : "smlevine",                |  "count" : 25054    |
|  "_id" : "robgeb",                  |  "count" : 23676    |

4. Number of total address with zip code and check how many of those are not in brooklyn.
```python
> db.brooklyn.find({"address.postcode":{"$gte":"11201", "$lte": "11256"}}).count()
> 290566
> db.brooklyn.find({"address.postcode":{"$exists": 1}}).count()
> 387592
> db.brooklyn.find({"address.postcode":{"$gte":"11201", "$lte": "11256"}}, {"tiger.zip_left":{"$gte":"11201", "$lte": "11256"}},{"tiger.zip_right":{"$gte":"11201", "$lte": "11256"}}).count()
> 290566
```
    - So the total number of address (including those pulled from tiger GPS) that have a valid zip code is 290566.

5. Update the zip code (delete all the documents that have out of boundary zip code)
```python
> db.brooklyn.remove({"address.postcode":{"$gt":"11256"}})
> WriteResult({ "nRemoved" : 80613 })
> db.brooklyn.remove({"address.postcode":{"$lt":"11201"}})
> WriteResult({ "nRemoved" : 16413 })
```
6. Explore how many of the tiger tag still need to be reviewed after updating zip code.
```python
> db.brooklyn.find({"tiger.reviewed":{"$exists": 1}}).count()
> 10106
> db.brooklyn.find({"tiger.reviewed":"yes"}).count()
> 251
```
    - If the tiger has been reviewed, the review tag is supposed to be deleted. There are 10106 tags that still have review value, but 251 of them have review value yes. So the total tags that still need to be reviewed are 9855.

7. Explore the top three amenities in brooklyn.
```python
> db.brooklyn.aggregate([{"$match":{'amenity':{$exists:1}}},{"$group":{'_id':'$amenity',"count":{"$sum":1}}},{"$sort":{"count":-1}},{"$limit":3}])
> { "_id" : "bicycle_parking", "count" : 2818 }
> { "_id" : "place_of_worship", "count" : 834 }
> { "_id" : "restaurant", "count" : 814 }

> db.brooklyn.aggregate([{"$match":{"amenity":{"$exists":1}, "amenity":"place_of_worship"}}, {"$group":{"_id":"$religion", "count":{"$sum":1}}}, {"$sort":{"count":-1}}, {"$limit":1}])
> { "_id" : "christian", "count" : 655 }

> db.brooklyn.aggregate([{"$match":{"amenity":{"$exists":1}, "amenity":"restaurant"}}, {"$group":{"_id":"$cuisine", "count":{"$sum":1}}}, {"$sort":{"count": -1}}, {"$limit":3}])
> { "_id" : null, "count" : 307 }
> { "_id" : "pizza", "count" : 47 }
> { "_id" : "italian", "count" : 43 }
```
    - **Conclusion: First, the large number of bicycle_parking is mainly due to the bike share system lunched by Mayor Bloomberg and Department of Transportation Commissioner Janette Sadik-Khan. Second, the dominant religion in brooklyn is unsurprising Christian. Finally, the documentation of restaurants in brooklyn needs to be improved. Various of cuisines are not categorized yet.**
    - Here are some examples of uncategorized restaurants:
```python   
<node changeset="6496165" id="1013624842" lat="40.716904" lon="-73.999527" timestamp="2010-11-30T17:39:27Z" uid="29040" user="chrismcnally" version="1">
<tag k="name" v="China Village Restaurant"/>
<tag k="amenity" v="restaurant"/>
<tag k="addr:street" v="Baxter Street"/>
<tag k="addr:housenumber" v="94"/>

<node changeset="11446248" id="1153146341" lat="40.7029557" lon="-74.0132642" timestamp="2012-04-29T03:27:09Z" uid="290680" user="wheelmap_visitor" version="4">
<tag k="name" v="Au Bon Pain"/>
<tag k="amenity" v="restaurant"/>
<tag k="wheelchair" v="limited"/>
```
### Other Thoughts
First, the quality of OpenStreetMap data for brooklyn is pretty good. Very little error in the format of addresses and zip code. However, all the data pulled from tiger GPS need to be fixed. It has some redundant fields, for example, some of the location has zip_left_1, zip_left_2, zip_right_1, zip_right_2 or more, but all those zip codes are exactly the same. I think it is reasonable to reduce them to one if they all have the same value.

Second, as more people moving to brooklyn recent years, the variety and number of restaurants increased enormously. Thus it definitely needs to be updated to provide valuable information for the economic of brooklyn.

Finally, comparison of the development of biking system in brooklyn with the traffic situation and health situation can be a very interesting project to dive in. It might help us gain valuable insight into the impact of biking system on local economy. 
