---
layout: default
title:  "Cheap & Easy scraping using Scrapy, Docker & Azure Container Instances"
date:   2022-04-25 11:33:27 +0100
categories: azure scraping container instances docker
description: "In this blog post I am going to show you how you can create a scraper using Python, Scrapy, Docker and deploy this solution to Azure Container Instances." 
---

# Cheap & Easy scraping using Scrapy, Docker & Azure Container Instances

Hi all,

In this blog post I am going to show you how you can create a scraper using Python, Scrapy, Docker and deploy this solution to Azure Container Instances.

Azure Container Instance is a service which you can use to easily run docker containers. These instances are billed per second, so to save money we will configure the service to shut down after it has finished. What's also nice about ACI is that your outbound IP frequently changes, so this might save the need for using a proxy, and save some more money.

Finally we will trigger the scraper to run every day using a Azure Logic App, and store all our data using CosmosDB (MongoDB API). The end result is a pretty cheap and simple scraper, which starts according to our schedule. 

### Requirements
- Python
- Docker 
- Scrapy project (You could use this [Hackernews Scrapy project](https://github.com/geekan/scrapy-examples/blob/master/hacker_news/hacker_news/spiders/spider.py) to build this container.


#### Step one, building the scraper
I will not go very in depth on Scrapy here, if you go to [this repository](https://github.com/geekan/scrapy-examples) you can find many examples. I will use a modified version of the hacker news sample for this tutorial.

To write the data to the database, first we will setup CosmosDB using the portal. So create a new CosmosDB database, and select the MongoDB API. Copy the connection string from CosmosDB and keep it somewhere, we're not going to bother with Azure Keyvault for this demonstration (but you should use it if you take this into production).

In order to be able to write our results we will slightly alter the sample project, because we're going to start the scraper from a python script (scrapeScript.py) instead of using the scrapy command.

scrapeScript.py:
```
from spiders import MySpider
from pipelines import MyPipeline

from pymongo import MongoClient

import scrapy
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings

client = MongoClient('MyDBConnectionString')
mydb = client["MyDB"]
mycol = mydb["MyCollection"]
settings = get_project_settings()

#this is where we start the scraper 
def scrape():
    process = CrawlerProcess(settings)
    process.crawl(MySpider)
    process.start() # the script will block here until the crawling is finished

    #and finally we get all the items and write them to the database.
    mycol.insert_many(MyPipeline.items)

scrape()
```

Our scrapy pipeline looks like this: 
pipelines.py:
```
class FundaPipeline(object):
    items=[]
    def process_item(self, item, spider):
        self.items.append(item)
        return item
```



#### Step two: building & deploying the container
Lets setup our cloud infrastructure first. We'll createa a Azure Container Registry first. So go create that using the portal and copy the container registry address (xxx.azurecr.io), username & password somewhere - we'll need this shortly...

The dockerfile, which docker will use to build the container, looks like this:

```
# Start from python slim-buster docker image, change python version as desired.
FROM python:3.7-buster
RUN apt-get update
# Copy files to working directory
COPY ./hacker_news /app/
WORKDIR /app 

# Install your dependencies from requirements.txt
RUN pip install -r requirements.txt

# Run scrapy - the container will exit when this finishes!
CMD python script.py
```

Now you can use the following command to build the container and tag it as victor/myScraper:v1.0.1
```docker build . --no-cache -t victor/myScraper:v1.0.1```
<br/>

To debug our container we can use the following command:
```docker run -it victor/myScraper:v1.0.1```
<br/>

The next command links our previously tagger docker image to our image which we are going to push to the Azure Container Registry.
```docker tag victor/myScraper:v1.0.1 mycontainerregistry.azurecr.io/myScraper:v1.0.1```
<br/>

And finally this command pushes our image to the Azure Container Registry.
```docker push mycontainerregistry.azurecr.io/myScraper:v1.0.1```      
<br/>

Okay so we have now uploaded our image to Azure Container Registry. The next step is for us to create a container instance using that image.  


The command which we will use to create a Azure Container Instance and link it to our ACR image is:
```az container create  --resource-group scraper --name myScraper --image mycontainerregistry.azurecr.io/myScraper:v1.0.1 --restart-policy OnFailure --memory 0.5```
<br/>

As you can see from this command, you are required to provide the resource group name which your container instance will reside it. We're also giving it a name and linking it to our previously uploaded image hosted in our Azure Container Registry. The restart policy ensures that when the container finishes, it will not start again. In practice this means that after the scraper has ran and written everything to the database, it will shut down. 

After running the command you can check the status of your container instance easily through the Azure Portal.

#### Step three: starting our scraper according to a schedule
Now for the last cherry on top we're going to setup a logic app to start this container every day. Create a _serverless_ logic app using the portal. Serverless ensures that we are only billed when the logic app is fired, and follow the next steps:

1. Create an empty logic app using the designer
2. Search for "schedule" and select a time which suits your - and the site which you are scraping - needs. Maybe scraping during the busiest hours of a website isn't so nice? 
3. Add a new step and search for "Container". Select Azure Container Instances.
4. Now search for "start" and select "start containers in a container group".
5. You may be prompted to sign in again now. 
6. Select your subscription, resource group & container name.

Et voila, thats that. Your scraper will be started according to your schedule, and stop again when you're finished. You can make changes and deploy changes by following these four steps again:

1. Docker build
2. Docker tag local:azurecr
3. Docker push 
4. az container create (with same name, but change the image, so it will not deploy again!)

I hope you found this useful!
Victor

{% include share-buttons.html %}



