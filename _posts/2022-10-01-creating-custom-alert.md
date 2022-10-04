---
title: Elastic stack - Use 3rd party threat intelligence as Reference data
date: 2022-10-01 10:00:00 +7
categories: [elasticsearch]
tags: [elasticsearch, abuseipdb, 3rd party threat intel]
---
This post is made possible by collaborating with [Juan Matthew](github.com/juan-matt). Go check him out! ;)

This is the second part of the elasticsearch blog. In the first part, I discuss how to setup your parent server and child server using elasticsearch, Kibana, and elasticagent.  
If you haven't setup yours, then I recommend to give it a read first. 

In this post, I'll show you how to make a custom alert using data from 3rd party (In this blog I'm going to use [abuseipdb](https://abuseipdb.com)) threat intel that hasn't been supported by elastic (yet).  

### How to (Overview)
In order to use 3rd party threat intelligence as index pattern that later can be correlated with your logs, we should be able to: 
1. Gather threat intelligence from 3rd party, ideally using the provided API by the 3rd party threat intelligence.
2. Process the threat intelligence into JSON format that satisfy elasticsearch requirement so that it can be read properly by elasticsearch

There are a few ways to create custom alert (or more spesifically index in elasticsearch lingo). However, I'll show how to use the elasticsearch API. 

### Prerequisite:
1. Setup elasticsearch on parent server. (Kibana is optional but highly recommended)
2. Terminal access to your parent server
3. Since I'm going to use abuseipdb, I need a free API key from [abuseipdb](https://abuseipdb.com) (only need to register), you can use any 3rd party threat intelligence to your liking as long as you follow the How to (Overview) section

> Note: If you follow my last elasticsearch post, then, you should have elastic password setup. We're gonna use curl a lot so to make it easier, set an elastic password as shell variable and while we're at it, set an abuseipdb API as shell variable by running

```terminal
elastic_password="your-elastic-password"

abuseipdb_API_key="your-api-key"
``` 

> If you don't have password enabled on your elasticsearch, then just ignore the creation of "elastic-password" variable 


### Pull data from abuseipdb 

To pull data containing IP that are considered malicious from abuseipdb, run the command below. You can read more about abuseipdb API [here](https://docs.abuseipdb.com/#introduction).

```bash
curl -G https://api.abuseipdb.com/api/v2/blacklist -d confidenceMinimum=90 -H "Key: $abuseipdb_API_key" -H "Accept: application/json" >> abuseipdb_response.txt
```

The command above will make an API call to abuseipdb, request a blacklist with the most reported IP address and store it in a text file called <code> abuseipdb_response.txt </code>  
An example response by abuseipdb:

```json
{"meta":{"generatedAt":"2009-02-20T07:17:54+00:00"},"data":[{"ipAddress":"123.456.789.012","countryCode":"ZZ","abuseConfidenceScore":100,"lastReportedAt":"2009-02-20T07:17:54+00:00"},{"ipAddress":"123.456.789.012","countryCode":"ZZ","abuseConfidenceScore":100,"lastReportedAt":"2009-02-20T07:17:54+00:00"},{"ipAddress":"123.456.789.012","countryCode":"ZZ","abuseConfidenceScore":100,"lastReportedAt":"2009-02-20T07:17:54+00:00"}
```

### Using string manipulation to satisfy elasticsearch format

Now we want to process this response to a format that elasticsearch can understand. Before that, we need to know what are the format that elasticsearch can understand.  
The format are as following:

1. Every line should be ended with a newline (\n) character.
2. Define the operation that we want to do. In this case, we want to add an IP to our blacklist. So we need to use the <code> {"index" :{}} </code> operation. 
3. After defining the operation that we want, it should be followed by the data that we want to do an operation with. 
4. The data given to elasticsearch should follow JSON format
5. Because later on we want to make a correlation with the blacklist, there should exist a "@timestamp" within the data. 

So, following the above spesification, our final product should be something like this:

```json
{ "index" : { } }
{"ipAddress":"123.456.789.012","countryCode":"ZZ","abuseConfidenceScore":100,"@timestamp":"2009-02-20T07:17:54+00:00"}
{ "index" : { } }
{"ipAddress":"123.456.789.012","countryCode":"ZZ","abuseConfidenceScore":100,"@timestamp":"2009-02-20T07:17:54+00:00"}
{ "index" : { } }
{"ipAddress":"123.456.789.012","countryCode":"ZZ","abuseConfidenceScore":100,"@timestamp":"2009-02-20T07:17:54+00:00"}
```
In reality, if we make an API call to abuseipdb, there'll be thounsands of these IP. It's basically impossible to edit these by hand. Instead we can make a script that can automate this. 

Below is a quick and dirty python script I wrote to process <code> abuseipdb_response.txt </code> to satisfy the format specified before. You can change the raw\_file variable to match your filename to your liking. However, in the script below, I use <code> processed_response.txt </code> as the final file.  
From now on I will assume that our final file name is the same.  

```python
################## Change your filename here ##################### 
raw_file = open("./abuseipdb_response.txt", "r")
final = open("./processed_response.txt", "w")
###################################################################

raw_response = raw_file.read()

index_data = raw_response.find("\"data\"")
processed_response = raw_response[index_data+8:]

processed_response = processed_response.replace("lastReportedAt", "@timestamp")

total_length = len(processed_response)
index_now = 0
index_delimiter = []
index_delimiter.append(0)
while index_now in range(total_length):
  if processed_response[index_now : index_now + 3] == "},{":
    index_delimiter.append(index_now + 1)
  index_now += 1

x = 0

while x < len(index_delimiter):
  opening_bracket = processed_response[index_delimiter[x]:].find("{")
  closing_bracket = processed_response[index_delimiter[x]:].find("}")
  final.write("{ \"index\" : { } }\n")
  final.write(processed_response[index_delimiter[x] + opening_bracket:index_delimiter[x] + closing_bracket + 1] + "\n")
  x += 1

final.close()
raw_file.close()
```

### Creating index

To create index and verify if the index successfully created. Run:

```bash
curl -u elastic:$elastic_password -X PUT "http://localhost:9200/blacklists"

curl -u elastic:$elastic_pass -X GET "http://localhost:9200/_cat/indices"
```

If the index is created successfully, you'll get a response
```text
{"acknowledged":true,"shards_acknowledged":true,"index":"blacklists"}
```

To further verify, see the output of GET request command from above and search for index named "blacklist". Example:
```
yellow open blacklists  ySXbEaJxQTaEIC3kaSBdoA 1 1    0     0    226b    226b
```

With that, we successfully created an index.


### Inputting processed data to elasticsearch

```bash
curl -u elastic:$elastic_password -H "Content-Type: application/x-ndjson" -X POST localhost:9200/blacklists/abuseipdb/_bulk --data-binary "@processed_response.txt"; echo "OK!"
```
Note: it's important that we use <code> --data-binary </code> to preserve the newline (\n) character instead of <code> -d </code> flag. 


### Viewing the blacklist index

First, open the burger menu on the top left, choose Stack Management -> Index Patterns -> Create Index Pattern.

![Open stack management menu](/public/elasticsearch/stack management.png)

![Create index pattern](/public/elasticsearch/create index pattern.png)

On the name field, type "blacklists*" (or whatever index name you created at step 2). For the timestamp field choose "@timestamp" then, click Create Index Pattern.

![Create index pattern](/public/elasticsearch/creating index pattern.png)

To view the blacklists data. Open the burger menu on the top left and click Discover and choose the blacklists* index pattern. 

![Create index pattern](/public/elasticsearch/viewing data.png)


### Done 😉️




























































































