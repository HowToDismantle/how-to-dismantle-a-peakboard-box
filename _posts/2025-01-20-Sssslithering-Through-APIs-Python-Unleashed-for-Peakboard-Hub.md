---
layout: post
title: Sssslithering Through APIs - Python Unleashed for Peakboard Hub
date: 2023-03-01 03:00:00 +0200
tags: api peakboardhub
image: /assets/2025-01-20/title.png
image_header: /assets/2025-01-20/title_landscape.png
read_more_links:
  - name: Cracking the code - Part I - Getting started with Peakboard Hub API
    url: /Cracking-the-code-Part-I-Getting-started-with-Peakboard-Hub-API.html
  - name: Cracking the code - Part II - Calling functions remotely
    url: /Cracking-the-code-Part-II-Calling-functions-remotely.html
downloads:
  - name: PeakboardHubAPIPlayground.py
    url: /assets/2025-01-20/PeakboardHubAPIPlayground.py
---
A couple of weeks ago we already started with an introduction to the Peakboard Hub API: [Getting started](/Cracking-the-code-Part-I-Getting-started-with-Peakboard-Hub-API.html). And also there's another article on how to use the API to [manage lists on the Peakbord Hub](/Cracking-the-code-Part-III-Reading-and-writing-lists-with-Peakboard-Hub-API.html)

This article is all about using the Peakboard Hub API in Python. Let's get right into it....

![image](/assets/2025-01-20/005.png)

## Authentification

Beside the typical import for API calls "import requests" we're also importing "pandas" for table formatting and "sys" to create some nice code in case we want to exit. The base URL depends on which Peakboard instance is supposed to be addressed. The sample shows the regular Hub Online. Please check the other articles on more information about to get the API key.

We submit the key in the header of the first call. When the calls was successful we can easily extract the access token from the response body and build a session instance with it. So in all later calls we don't need to worry about authentification as long as as we use the session instance. 

{% highlight python %}
import requests
import sys
import pandas

BaseURL = "http://api.peakboard.com";
APIKey = "XXX";

response = requests.get(BaseURL + "/public-api/v1/auth/token", headers={'apiKey': APIKey})

if response.status_code != 200:
    sys.exit("Unable to obtain a access token, so quitting") 

accesstoken = response.json()["accessToken"]
print("Succesfully authorized until " + response.json()["validUntill"])

mySession = requests.Session()
mySession.headers.update({"Authorization": "Bearer " + accesstoken})
{% endhighlight %}

## Get all lists

Obtaining all available lists from the Hub is a simple GET call. We just loop over the array that is returned in the response body's JSON string.

{% highlight python %}
# Get all lists

response = mySession.get(BaseURL + "/public-api/v1/lists")

if response.status_code != 200:
    sys.exit("Unable to obtain list information")

for item in response.json():
    print(f"Table found: {item['name']}")
{% endhighlight %}

![image](/assets/2025-01-20/010.png)

## Get list data

With a second simple GET call we receive the actual list data. The actual command is sent in the query (Name of table and sorting). With the result we make use of the "pandas" lib. The columns of the table and each record are turned into collections and just formatted by use of this external library with one single easy call "pandas.DataFrame(...)".

{% highlight python %}
# Get data of a list

response = mySession.get(BaseURL + "/public-api/v1/lists/list?Name=stockinfo&SortColumn=MaterialNo&SortOrder=Asc")

if response.status_code != 200:
    sys.exit("Unable to obtain list data")


columns = [col['name'] for col in response.json()['columns']]
items = [{entry['column']: entry['value'] for entry in item} for item in response.json()['items']]

print(pandas.DataFrame(items, columns=columns))
{% endhighlight %}

![image](/assets/2025-01-20/020.png)

## Create a record

Creating a new record is done through a POST call to /public-api/v1/lists/items. The sample code shows how to construct a body with the name of the list and the actual data to be added. The strcuture is so simple that it can be cosidered as self-explainary. When the call is succesful we can read the ID of the record from the JSON that is returned in the response.

{% highlight python %}
# create a new record

body = {
    "listName": "stockinfo",
    "data": {
        "MaterialNo": "0815",
        "Quantity": 5,
        "locked": False  
    }
}

response = mySession.post(BaseURL + "/public-api/v1/lists/items", json=body)

if response.status_code != 200:
    sys.exit("Unable to create a new record. Return is " + response.reason)

id = response.json()["addedItem"][0]["value"]
print("New record added under ID " + str(id))#
{% endhighlight %}

## Edit a record

Editing a record works similiar to creating a record but with a PUT command to /public-api/v1/lists/items. Beside the list name we must add the row ID of the data to be edited.

{% highlight python %}
# Edit a record

body = {
  "rowId": id,
  "listName": "stockinfo",
  "data": {
    "Quantity": 30
  }
}

response = mySession.put(BaseURL + "/public-api/v1/lists/items", json=body)

if response.status_code != 200:
    sys.exit("Unable to change a record. Return is " + response.reason)

print(f"Record with id {id} was changed")
{% endhighlight %}

## Delete a record

To complete the series we have a final look at deleting data by using the DELETE call to /public-api/v1/lists/items. We must provide the name of the list and ID to be deleted. That's all.

{% highlight python %}
# delete a record

body = {
  "listName": "stockinfo",
  "rowId": id
}

response = mySession.delete(BaseURL + "/public-api/v1/lists/items", json=body)

if response.status_code != 200:
    sys.exit("Unable to delete a record. Return is " + response.reason)

print(f"Record with id {id} was deleted")
{% endhighlight %}
