# OpenStreetMap Data Wrangling Process

## A Short Introduction on OpenStreetMap
OpenStreetMap (OSM) foundation is building free and editable map of the world, enabling the development of freely-reusable geospatial data. The data from OpenStreetMap is being used by many applications such as GoogleMaps, Foursquare and Craigslist.

OpenStreetMap data structure has four core elements: Nodes, Ways, Tags, and Relations

- Nodes are points with a geographic position stored as lon (longitude) and lan (latitude). They are used to show points on the map such as points of interest.
- Ways are ordered list of nodes, representing a polyline or a polyline if they make a closed loop. They are used to show streets, rivers, and area such as parks, lakes, etc.
- Relations are ordered list of nodes, ways and relations (called 'members') where each members can have a 'role' (a string). They are used to show the relation between nodes and ways such as restrictions on roads.
- Tags are pairs of Keys and Values. They are used to store metdata of the map such as address, type of building, or any sort of physical property. Tags are always attached to a node, way or relation and are not stand-alone elements.

To look at the map, or download your area of interest, you can visit http://www.openstreetmap.org website. 

For more information you can check their wiki which includes all the necessary information and documentation:
https://en.wikipedia.org/wiki/OpenStreetMap

Users can add points on the map, create relations of the nodes and ways, and assign properties such as address or type to them. The data can be stored in OSM format, and can be access in different formats. For the purpose of this project, I use the OSM XML format.
http://wiki.openstreetmap.org/wiki/OSM_XML

In this project, I will work with the raw data from an area. Since the data is put by different users, I suppose it can be quite messy; therefore, I will use cleaning and auditing functions to make the data look clean for analysis. I will export the data into CSV format and use this format to create tables in my SQLite database. Then I run queries on the database to extract information such as number of nodes and ways, most contributing users, top points of interest, etc. I will conclude the project by discussing benefits as well as some anticipated problems in implementing the improvement.

## Area Chosen for Analysis

For this project, I chose San Francisco area in the US. I chose this area as it is a point of interest for me with its big IT corporations; also, it is a place I want to travel to one day. I decided to download the file locally to my machine.  
https://mapzen.com/data/metro-extracts/metro/san-francisco_california/

The 'metro extracts' will provide the map of the metropolian area (i.e. where you can find more elements to work with)

The original file is about 1.01GB in size; however, I use a sample file about 50MB to perform my initial analysis on. Once I am satisfied with the code, I run it on the original file to create the CSV files for my database. 

The data analyzed in this Jupyter notebook is from the sample file to be able to show shorter results from my analysis. I have included all the functions here, as well as the function to create CSV files, in separate .py files in my repository. Using those you can run the code on the original file. 
https://github.com/Nazaniiin/OpenStreetMap_DataWrangling

## Exploring the Data a bit

Let's start going through the data, find its problems and clean them. First, we'll take a look into the dataset and parse through using ElementTree and extract information such as different types of elements (nodes,ways,etc.) in the OSM XML file.

Using ET.iterparse (i.e. iterative parsing) is efficient here since the original file is too large for processing the whole thing. So iterative parsing will parse the file as it builds it.  
http://stackoverflow.com/questions/12792998/elementtree-iterparse-strategy

In this code, I will iterate through different tags in the XML (nodes,ways,relations,member,...) and count them, put them in a dictionary with the key being the tag name.

```python
import xml.etree.cElementTree as ET
import pprint

OSMFILE = '/Users/nazaninmirarab/Desktop/Data Science/P3/Project/Submission2/san-francisco_california_sample.osm'

def count_tags(filename):
    tags= {}
    for event, elem in ET.iterparse(filename):
        if elem.tag not in tags.keys():
            tags[elem.tag] = 1
        else:
            tags[elem.tag] += 1
    
    pprint.pprint(tags)
    
count_tags(OSMFILE)
```
```
{'member': 2530,
 'nd': 281292,
 'node': 235744,
 'osm': 1,
 'relation': 283,
 'tag': 87071,
 'way': 27558}
 ```
I will do a bit more exploration on the data. In the OSM XML file, the 'tag' element has key-value pairs which contain information about different points (nodes) or ways in the map. I parse through this element using the following regular expressions:
- lower -> ^([a-z]|_)*$ : This matches strings that contain only lower case characters. I start at the beginning of the strong and match between zero to unlimited times a character in range 'a' to 'z'. This regular expression also covers the underscore '_' character. 

- lower_colon -> ^([a-z]|_)*:([a-z]|_)*$ : This matches strings which contain lower case characters but also have the colon ':' character e.g. addr:street is one type of tag which specifies a street name. 

- problemchar -> [=\+/&<>;\'"\?%#$@\,\. \t\r\n] : This matches tags with problematic characters specified in the regex pattern. 

Take this section of the map as an example:

    <node changeset="30175357" id="358830340" lat="37.6504905" lon="-122.4896963" timestamp="2015-04-12T22:43:37Z" 
        uid="35667" user="encleadus" version="4">
		<tag k="name" v="Ocean Shore School" />
		<tag k="phone" v="+1 650 738 6650" />
		<tag k="amenity" v="school" />
		<tag k="website" v="http://www.oceanshoreschool.org/" />
		<tag k="addr:city" v="Pacifica" />
		<tag k="addr:state" v="CA" />
		<tag k="addr:street" v="Oceana Boulevard" />
		<tag k="gnis:created" v="04/06/1998" />
		<tag k="addr:postcode" v="94044" />
		<tag k="gnis:state_id" v="06" />
		<tag k="gnis:county_id" v="081" />
		<tag k="gnis:feature_id" v="1785657" />
		<tag k="addr:housenumber" v="411" />
	</node>
    
This node tag has 13 tag elements inside it. There are multiple keys that have the ':' character in them, so they fall under the 'lower_colon' regular expression. keys like name, phone, and amenity will fall under the 'lower' regular expression. There are no problematic characters in this specific node.

```python
import xml.etree.cElementTree as ET
import pprint
import re

OSMFILE = '/Users/nazaninmirarab/Desktop/Data Science/P3/Project/Submission2/san-francisco_california_sample.osm'

lower = re.compile(r'^([a-z]|_)*$')
lower_colon = re.compile(r'^([a-z]|_)*:([a-z]|_)*$')
problemchars = re.compile(r'[=\+/&<>;\'"\?%#$@\,\. \t\r\n]')


def key_type(element, keys):
    if element.tag == "tag":
        for tag in element.iter('tag'): #iterating through the tag element in the XML file
            k = element.attrib['k'] #looking for the tag attribute 'k' which contains the keys
            if re.search(lower, k):
                keys['lower'] += 1
            elif re.search(lower_colon, k):
                keys['lower_colon'] += 1
            elif re.search(problemchars, k):
                keys['problemchars'] += 1
            else:
                keys['other'] += 1
                
    return keys

def process_map(filename):
    keys = {"lower": 0, "lower_colon": 0, "problemchars": 0, "other": 0}
    for _, element in ET.iterparse(filename):
        keys = key_type(element, keys)

    pprint.pprint(keys)
    
process_map(OSMFILE)
```
```
{'lower': 51867, 'lower_colon': 33909, 'other': 1281, 'problemchars': 14}
```
I now want to collect some information about the users contributed to the OpenStreetMap data for San Francisco area. I want to calculate the number of unique users. 

To find the users, we need to look through the attributes of the node, way and relation tags. The 'uid' attribute is what we need to count.

    <node changeset="30175357" id="358830340" lat="37.6504905" lon="-122.4896963" timestamp="2015-04-12T22:43:37Z" 
    uid="35667" user="encleadus" version="4">

```python
OSMFILE = '/Users/nazaninmirarab/Desktop/Data Science/P3/Project/Submission2/san-francisco_california_sample.osm'

def process_map(filename):
    users = set()
    for _, element in ET.iterparse(filename):
        if element.tag == 'node' or element.tag == 'way' or element.tag == 'relation':
                userid = element.attrib['uid']
                users.add(userid)

    print len(users)
    
process_map(OSMFILE)
```
## How the Project Proceeds

The rest of the project consists of:
- Auditing street names
- Auditing postcodes
- Preparing the data for the database
- Creating tables in the database
- Running queries on the data
- Conclusions

The detailed case study includes information about all snippets of codes, as well as outputs from the query. You can find the complete report from this repository from [CaseStudy for OpenStreetMap](https://github.com/Nazaniiin/OpenStreetMap_DataWrangling/blob/master/CaseStudy_OpenStreetMap.ipynb) 

## The Files in This Repository

The Jupyter notebook gives a complete exaplanation regarding the project, the code, and queries. To be able to run the code locally, you need to have the following files:
- **shaping_csv.py** : The main function where the CSV files get created from the OSM data
	- Remember to check the imports and have the necessary libraries installed
- **audit.py** : This script contains the auditing and cleaning functions for editing street names and postcodes.
	- Store this file in the same directory as the shaping_csv.py file as shaping_csv calls the functions in audit.py 
- **creating_database.py** : This script create your database and tables, then insert the data from nodes, nodes_tags, ways, ways_tags and ways_nodes in corresponding tables
	- The information to insert the data comes from CSV files that got created while running shaping_csv.py script
- **query_on_database.py** : This script contains all the SQL commands I ran on the data. All the script with their output are available from the [CaseStudy for OpenStreetMap](https://github.com/Nazaniiin/OpenStreetMap_DataWrangling/blob/master/CaseStudy_OpenStreetMap.ipynb)
- **san-francisco_california_sample.osm** : This file contains a sample data of size 50MB from the original San Francisco database. The original file size for San Francisco database is around 1GB.


