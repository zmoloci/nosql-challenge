# Module 12 Challenge

### Instructions

```The UK Food Standards Agency evaluates various establishments across the United Kingdom, and gives them a food hygiene rating. You've been contracted by the editors of a food magazine, Eat Safe, Love, to evaluate some of the ratings data in order to help their journalists and food critics decide where to focus future articles.```

## Dependencies
[pymongo 4.3.3](https://pypi.org/project/pymongo/) </br>
[Data pretty printer - pprint](https://docs.python.org/3/library/pprint.html) </br>
[Pandas](https://pandas.pydata.org/pandas-docs/stable/) </br>
[MatPlotLib 3.7.1](https://matplotlib.org/stable/tutorials/introductory/pyplot.html) </br>
```
from pymongo import MongoClient
from pprint import pprint
import pandas as pd
import matplotlib.pyplot as plt
```

## Part 1: Database and Jupyter Notebook Set Up

UK Food Standards Agency data was imported to a newly created local database (uk_food) in order to update, explore and analyze the data within: </br>

In Terminal:
```
mongoimport --type json -d uk_food -c establishments --drop --jsonArray establishments.json
```

In Jupyter Notebook:

```
mongo = MongoClient(port=27017)

# confirm that our new database 'uk_food' was created
print(mongo.list_database_names())

db = mongo['uk_food']

print(db.list_collection_names())

establishments = db['establishments']

# review a document in the establishments collection
pprint(db.establishments.find_one())
```

## Part 2: Update the Database

1. Dictionary for new restaurant "Penang Flavours" was created and inserted into the 'establishments' collection: </br>
```
penangFlavours = {
    "BusinessName":"Penang Flavours",
    "BusinessType":"Restaurant/Cafe/Canteen",
    "BusinessTypeID":"",
    "AddressLine1":"Penang Flavours",
    "AddressLine2":"146A Plumstead Rd",
    "AddressLine3":"London",
    "AddressLine4":"",
    "PostCode":"SE18 7DY",
    "Phone":"",
    "LocalAuthorityCode":"511",
    "LocalAuthorityName":"Greenwich",
    "LocalAuthorityWebSite":"http://www.royalgreenwich.gov.uk",
    "LocalAuthorityEmailAddress":"health@royalgreenwich.gov.uk",
    "scores":{
        "Hygiene":"",
        "Structural":"",
        "ConfidenceInManagement":""
    },
    "SchemeType":"FHRS",
    "geocode":{
        "longitude":"0.08384000",
        "latitude":"51.49014200"
    },
    "RightToReply":"",
    "Distance":4623.9723280747176,
    "NewRatingPending":True
}
```

```
# Insert the new restaurant into the collection
establishments.insert_one(penangFlavours)
```
2. Query the collection to discern the BusinessTypeID value for "Restaurant/Cafe/Canteen":

```
query = {'BusinessType':"Restaurant/Cafe/Canteen"}
fields = {'BusinessTypeID':1, 'BusinessType':1}

results = list(establishments.find(query,fields).limit(1))
pprint(results)
```

3. "Penang Flavours" was updated with the appropriate BusinessTypeID:

```
filter = {"BusinessName":"Penang Flavours"}
newvalues = { "$set": { 'BusinessTypeID': 1}}

establishments.update_one(filter,newvalues)
```

The establishment document for "Penang Flavours was checked to confirm that the value was correctly updated:

```
result3 = establishments.find_one({"BusinessName":"Penang Flavours"})
pprint(result3)
```

4. The documents for the Dover Local Authority were then queried and removed:

```
# Find how many documents have LocalAuthorityName as "Dover"
query = {'LocalAuthorityName':"Dover"}


results = list(establishments.find(query))
print(len(results))

# Delete all documents where LocalAuthorityName is "Dover"
establishments.delete_many({'LocalAuthorityName':"Dover"})
```
The collection was then queried to confirm that Dover establishments were successfully removed:
```
query = {'LocalAuthorityName':"Dover"}


results = list(establishments.find(query))
print(len(results))


# Check that other documents remain with 'find_one'
establishments.find_one()
```

5. The latitude and longitude values were converted from string to decimals:

```
establishments.update_many({},[ {'$set': { "geocode.longitude" : {'$toDouble': "$geocode.longitude"},
                                          "geocode.latitude" : {'$toDouble': "$geocode.latitude"}
                                          }
                                }   ]
                        )
                        
```
The coordinates were then checked to confirm that they were no longer strings:
```
fields = {'geocode.longitude': 1, 'geocode.latitude': 1}

results = establishments.find({},fields)

for i in range(10):
    pprint(results[i])

```


## Part 3: Exploratory Analysis

1. The collection was queried to determine which establishments have a hygiene score equal to 20?
```
query = {'scores.Hygiene': 20}

results = list(establishments.find(query))

# Use count_documents to display the number of documents in the result
hygieneTwenty = establishments.count_documents({'scores.Hygiene':20})
print(hygieneTwenty)

# Display the first document in the results using pprint
pprint(results[0])
```
The results were then converted to a Pandas Dataframe and the first 10 rows were displayed:
```
hygieneTwenty_df = pd.DataFrame(results)
# hygieneTwenty_df.head()
# Display the number of rows in the DataFrame
len(hygieneTwenty_df)
# Display the first 10 rows of the DataFrame
hygieneTwenty_df.head(n=10)
```

2. London establishments with 'RatingValue' greater than or equal to 4 were displayed:

```
query = {"RatingValue":{'$in':["4","5",4,5]}, "LocalAuthorityName": {'$regex':'London'}}
results = list(establishments.find(query))
# Use count_documents to display the number of documents in the result
fourOrBetter = establishments.count_documents({"RatingValue":{'$in':["4","5",4,5]}, "LocalAuthorityName": {'$regex':'London'}})
print(fourOrBetter)

# Display the first document in the results using pprint
pprint(results[0])
```
The results were then converted to a Pandas Dataframe and the first 10 rows were displayed:
```
fourOrBetter_df = pd.DataFrame(results)
# Display the number of rows in the DataFrame
print(len(fourOrBetter_df))
# Display the first 10 rows of the DataFrame
fourOrBetter_df.head(10)
```


3. The collection was queried to determine the 5 establishments with a "RatingValue" of 5 that were located closest to "Penang Flavours" and also had the lowest hygiene score:

```
degree_search = 0.00155
# I reduced the value above in order to find the closest establishments without adding 'near' functionality
# This reduced value provided exactly 5 establishments with a Hygiene score = 0
# When the 'degree_search' was set to 0.01 as suggested, more than 5 establishments were returned 
# with 'Hygiene' = 0, so the first 5 documents would not technically be the "nearest to the new restaurant".

latitude = 51.49014200
longitude = 0.08384000

# Query the data and include just the establishments that have a rating value of 5 or "5". Also only
# return establishments within a certain square around "Penang Flavours". This area has dimensions of
# 2 x degree_search by 2 x degree_search and is centered on "Penang Flavours"
query = {"geocode.latitude": { '$gte' : (latitude - degree_search), '$lte' : (latitude + degree_search) },
         "geocode.longitude": { '$gte' : (longitude - degree_search), '$lte' : (longitude + degree_search) },
         "RatingValue":{'$in':["5",5]}}
        
# sort values to from lowest to highest Hygiene score
sort = [('scores.Hygiene', 1)]

# Print the results (with limit 5)
results = list(establishments.find(query).sort(sort).limit(5))

# pretty print results
pprint(results)

```
As noted in the comments above, initially the "degree_search" value was set to 0.01 as per the starter code provided, though it was </br>
evident that when the data was sorted by hygiene and the results were limited to 5 documents, the first 5 documents displayed were not </br>
the closest restaurants to "Penang Flavours". </br>
In order to establish the correct "degree_search" value to return the 5 closest establishments to "Penang Flavours", the limit was temporarily </br>
increased to 10 and the "degree_search" value was iteratively reduced until only 5 establishments with a Hygiene score = 0 remained. </br>

</br>

The results were then added to a dataframe and displayed:
```
topfivecleaneats_df = pd.DataFrame(results)
top_five_clean_eats_near_penang_df
```
4. A data pipeline was created to determine how many establishments with hygiene score = 0 operate in each Local Authority:

```
# 1. Matches establishments with a hygiene score of 0
match_query = {'$match': {'scores.Hygiene':0}}
# 2. Groups the matches by Local Authority
# add 1 to each Local Authority for each establishment with a Hygiene score = 0
group_query = {'$group': {'_id': {'LocalAuthorityName': '$LocalAuthorityName'}, 'count': {'$sum': 1}}}
# 3. Sorts the matches from highest to lowest
# sorted Local Authority in descending order by count of establishments with Hygiene = 0
sort_values = {'$sort': {'count':-1}}

pipeline = [match_query,group_query,sort_values]

results = list(establishments.aggregate(pipeline))
# Print the number of documents in the result
print("Number of Local Authorities in result: ", len(results))
```
Then the top 10 Local Authorities, by count of Hygiene score = 0 were displayed:
```
# Print the first 10 results
pprint(results[0:10])

```
All results were then added to a Pandas Dataframe and the top 10 rows were displayed:

```
aggregated_df = pd.json_normalize(results)
# Display the number of rows in the DataFrame
len(aggregated_df.index)
# Display the first 10 rows of the DataFrame
aggregated_df.head(10)
```


# References
UK Food Standards AgencyLinks to an external site. (2022). UK food hygiene rating data API. https://ratings.food.gov.uk/open-data/en-GBLinks to an external site.. Contains public sector information licensed under the Open Government Licence v3.0Links to an external site.
Accessed Sept 9, 2022 and Sept 12, 2022 with the establishment settings as follows: longitude=51.5072, latitude=-0.1276, maxdistancelimit=4567, pagesize=10000, sortoptionkey=distance, pagenumber=(1,2,3,4,5,6,7,8).
