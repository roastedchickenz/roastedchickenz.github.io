---
title: Elastic stack - Setting up elasticsearch, kibana, and elasticagent 
date: 2022-09-30 10:00:00 +7
categories: [elasticsearch]
tags: [elasticsearch, setup]
---
This post is made possible by collaborating with [Juan Matthew](github.com/juan-matt). Go check him out! ;)

In this posts, I will try to guide you to setup a parent server with 1 child server (to add more than 1 child server, simply repeat the steps to add add a child server) using elasticsearch, kibana, and elastic agent.


And in the following posts, I will try to guide you to use any 3rd party threat intelligence to your liking as reference data that can be correlated with logs.  

Before we begin, this is an overview of the setup that we'll do. 

![topology overview](/public/elasticsearch/topology overview.png)


## Setting up parent node

It's always a good idea to read the [documentation](https://www.elastic.co/guide/en/elastic-stack/7.17/installing-elastic-stack.html) since they are the one who writes the code. So they should be the one who knows the most about the application. 


From the documentation, they recommend that we install the elastic stack in this order: elasticsearch -> kibana -> elasticagent  So we'll do just that!

### Installing and configuring elasticsearch

I literally copy-pasted these commands below from the documentation.  
If you're using init then you should adjust accrodingly.  
If you're using systemd, just follow along 😉️

```terminal
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

sudo apt-get install apt-transport-https

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list

sudo apt-get update && sudo apt-get install elasticsearch

sudo /bin/systemctl daemon-reload

sudo /bin/systemctl enable elasticsearch.service

sudo systemctl start elasticsearch.service

```
Note: Don't forget to properly configure your firewall.  Elasticsearch by default listens on port 9200

After that, edit the config <code> elasticsearch.yml </code> file (usually located in /etc/elasticsearch)

```yaml
....
# ---------------------------------- Network -----------------------------------
# By default Elasticsearch is only accessible on localhost. Set a different
# address here to expose this node on the network:
#
network.host: 0.0.0.0
#
# By default Elasticsearch listens for HTTP traffic on the first free port it
# finds starting at 9200. Set a specific HTTP port here:
#
http.port: 9200
#
# For more information, consult the network module documentation.
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when this node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
discovery.type: single-node
discovery.seed_hosts: ["0.0.0.0"]
#
# Bootstrap the cluster using an initial set of master-eligible nodes:
#
#cluster.initial_master_nodes: ["node-1", "node-2"]
#
# For more information, consult the discovery and cluster formation module documentation.
....

```

After editing & saving elasticsearch.yml, restart elasticsearch

```terminal
sudo systemctl restart elasticsearch
```

You might want to verify that now we're able to access elasticsearch by running:

```terminal
curl http://xxx.xxx.133.123:9200
```
You should get an answer similar to this:
```text
{
  "name" : "parent-elastic",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "2-K4MpDPRcaWVEiA14o15g",
  "version" : {
    "number" : "7.17.6",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "f65e9d338dc1d07b642e14a27f338990148ee5b6",
    "build_date" : "2022-08-23T11:08:48.893373482Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```
>If you receive an error similiar to "connection refused", please check your  elasticsearch.yml file again or your firewall settings or try reinstalling elasticsearch.

You think we're done configuring elasticsearch? haha you're wrong! Since we want to use the "Security" function from Kibana, we MUST configure basic authentication aka passwords.  

We need to enable the "Security" function first. Edit your config file <code> elasticsearch.yml </code> file (again) and add the following line;where you add these config generally doesn't really matter but I usually add them near the bottom:
```yaml
xpack.security.enabled: true
xpack.security.authc.api_key.enabled: true
``` 
Then, we need to generate a password to authenticate.  
To do so, go to <code> /usr/share/elasticsearch/bin </code> and execute:
```terminal
./elasticsearch-setup-passwords auto
```
Take note of the outputted passwords, we'll need them while configuring kibana. 

### Installing and configuring Kibana

Since we've added elastic's GPG key and repo from installing elasticsearch earlier, all we need to do to install Kibana is:
```terminal
sudo apt-get update && sudo apt-get install kibana

sudo /bin/systemctl daemon-reload

sudo /bin/systemctl enable kibana.service

sudo systemctl start kibana.service
```
and edit config file <code> kibana.yml </code> file (usually located in /etc/kibana).  
We'll use the previously generated passwords here.  Alternatively, you can use [kibana-keystore](https://www.elastic.co/guide/en/kibana/7.17/secure-settings.html) to help manage and secure credentials, but for the sake of simplicity, I won't show you how to use kibana-keystore here.   
Instead, I copy-pasted the password to the config file. If you want to deploy this to production, you should really consider using [kibana-keystore](https://www.elastic.co/guide/en/kibana/7.17/secure-settings.html)
```yaml
....
# Specifies the address to which the Kibana server will bind. IP addresses and host names are both valid values.
# The default is 'localhost', which usually means remote machines will not be able to connect.
# To allow connections from remote users, set this parameter to a non-loopback address.
server.host: "0.0.0.0"
....

elasticsearch.username: "elastic"
elasticsearch.password: "xxxxxxxxx" #your previously generated passwords
........
```

Note: Don't forget to properly configure your firewall.  Kibana by default listens on port 5601.

After editing & saving kibana.yml , restart kibana

```terminal
sudo systemctl restart kibana
```

Verify that now we're able to access kibana by accessing http://xxx.xxx.133.123:5601 on your browser.

![Kibana](/public/elasticsearch/kibana.png)

You can login using the elastic:xxxxxx previously generated.  

### Installing elasticagent and adding a fleet (parent) server
Ideally, you want the server who has kibana that becomes the fleet (parent) server for easier data access through UI. So, in this case I'll be using http://xxx.xxx.133.123 as the parent server because it has Kibana installed. I would recommend exploring Kibana on your own to see its full capability.

Anyway, to add a fleet server, go to Fleet menu.  

!["Fleet" menu can be accessed through Burger Menu on top left, look for "Management" section and click "Fleet"](/public/elasticsearch/fleet-menu.png)

Download, extract, and run elastic-agent as fleet server (See command below). 

![adding a fleet(parent) server](/public/elasticsearch/adding-fleet-server.png)
I use the Quick Start option here. If you want to deploy this to production, you should consider using your own certificate.  
For "Fleet Server hosts", this ideally should be the server who has Kibana installed. This server will be the parent server. Fleet server use port 8220 by default.
Then, click "Generate Service Token".

Lastly, I literally copy-pasted the command given by the instruction on the Fleet menu. 

```terminal
wget https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-7.17.6-linux-x86_64.tar.gz

tar -xzf elastic-agent-7.17.6-linux-x86_64.tar.gz

cd elastic-agent-7.17.6-linux-x86_64.tar.gz

sudo ./elastic-agent install --fleet-server-es=http://localhost:9200 --fleet-server-service-token=your-token --fleet-server-policy=policy-id --fleet-server-insecure-http
```
check if parent server successfully added
![check if parent server successfully added](/public/elasticsearch/parent-added.png)

Next, we want to make a new policy for our child server which we'll setup in a second.  

![creating policy](/public/elasticsearch/creating-policy.png)

Go to your newly created policy. Click Actions -> Add Agent.  
You'll see a set of instruction and command. Continue to Setting up child node section.  

## Setting up child node
### Install elasticagent
To setup a child node, we must enroll our child server to the parent server. Go to your child server. Download, extract and run command provided by Kibana.  
Since I'm using "Quick Start" option which utilizes self-signed ceritificate, I need to use the <code> --insecure </code> flag. 

```terminal
wget https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-7.17.6-linux-x86_64.tar.gz

tar -xzf elastic-agent-7.17.6-linux-x86_64.tar.gz

cd elastic-agent-7.17.6-linux-x86_64.tar.gz

sudo ./elastic-agent install --url=http://xxx.xxx.133.123:8220 --enrollment-token=your-token --insecure
```
check if child server successfully added
![check if child server successfully added](/public/elasticsearch/child-added.png)


## Done 😉️

On the next post, I will try to guide you how to use 3rd party threat intelligence as reference data that can be correlated with your logs.































