---
layout: default
title:  "How to run a (Python) Azure Function as a Docker container"
date:   2022-04-04 11:33:27 +0100
categories: azure customvision
description: "In this blog post I will walk you through the steps required to deploy your Azure Function as a Docker container. In this case the function was dockerized in order to be able to use a specific 
ODBC driver." 
---

## Introduction
In this blog post I will walk you through the steps required to deploy your Azure Function as a Docker container. In this case the function was dockerized in order to be able to install a Intersystems Cache ODBC driver (a form of proprietary database), which you can't do in a normal app service environment. 

To get this project to work in Azure you will roughly need to do two things:
1. Build the container
2. Deploy the container

#### Requirements
If you want to follow through the steps of this article, I will assume you have installed:
- VS Code
- VS Code Azure Functions extension
- Docker (duh)
- AZ Cli Tools


## Building the container
First we will need to create an Azure Functions project. We will use the VS Code Azure Function extension to generate a Python 3.6 HTTP Trigger function. 
Try the container locally (press F5 in VS code) to make sure everything is working correctly.
Now the next step is to build the container. We will create a dockerfile in the root of the functions project. The docker image provided by Microsoft(mcr.microsoft.com/azure-functions/python:3.0-python3.6) is built on top of Debian 10, a popular Linux distribution. Because we run the function in a docker container, we will be able to install the ODBC driver as we would on a 'normal' Linux server. I have added the dockerfile I used to build the Azure Function docker container below:
``` 
To enable ssh & remote debugging on app service change the base image to the one below
# FROM mcr.microsoft.com/azure-functions/python:3.0-python3.6-appservice
FROM mcr.microsoft.com/azure-functions/python:3.0-python3.6

ENV AzureWebJobsScriptRoot=/home/site/wwwroot \
    AzureFunctionsJobHost__Logging__Console__IsEnabled=true

# Add ODBC Driver requirements 
RUN apt-get update \
 && apt-get install unixodbc -y \
 && apt-get install unixodbc-dev -y \
 && apt-get install freetds-dev -y \
 && apt-get install freetds-bin -y \
 && apt-get install tdsodbc -y \
 && apt-get install --reinstall build-essential -y

# add pyodbc requirements
RUN apt-get install python-pyodbc -y

#copy the driver to container
COPY /Driver /usr/lib/intersystems/odbc

#run install script
RUN /usr/lib/intersystems/odbc/ODBCinstall

#copy the odbc configuration file to the container
COPY odbc.ini /etc/intersystemsodbc.ini

#make the odbc library aware that we have added a new driver
RUN odbcinst -i -d -f /etc/intersystemsodbc.ini

#symlink the odbc driver, not sure if its required
RUN ln -s /usr/lib/x86_64-linux-gnu/libodbccr.so.2.0.0 /usr/lib/x86_64-linux-gnu/odbc/libodbccr.so

#copy the Azure Functions project to the docker image
COPY . /home/site/wwwroot

#install the python libraries as you would usually do
RUN cd /home/site/wwwroot && pip install -r requirements.txt
```

**Build**

When you issue the docker build command, it will run through the dockerfile to install the requirements and perform the steps. In this case I am also tagging the container as myacc/azurefunctionsimage:v0.0.2, to identify this particular build.

```docker build --progress=plain --tag "myacc/azurefunctionsimage:v0.0.2" ```

<br/><br/>

**Run**

The Docker Run command starts up the container which we built in the previous step. By adding the -p (port) parameter, we expose the port 80 on the docker container, to respond to requests on the 8080 port. 

```docker run -p 8080:80 -it "myacc/azurefunctionsimage:v0.0.2"```

<br/><br/>

**Try container**

Now if we want to try to send requests to the dockerized Azure function, we can try the following url:
http://localhost:8080/api/HttpExample?name=Functions

<br/><br/>

**Inspect container**

We can still take a look inside the container and see how everything is configured. Taking a look inside can be very useful for debugging purposes.

```
1. docker ps
2. docker exec -it "FIRSTCHARSOFCONTAINER" /bin/bash
```

Now you can explore the container using the command line! When building the docker container, I used this tool to check whether all files were in the correct place.

## Deploying it to Azure
In order to be able to deploy our solution to Azure, the first step is to create a Azure Container Registry. We will use this to store the container image we built.

This bicep snippet creates a container registry in Azure. 

```
resource containerRegistry 'Microsoft.ContainerRegistry/registries@2021-06-01-preview' = {
  //Name needs to be unique, so change this!!
  name: 'cachetwofunctionacr'
  location: 'westeurope'
  sku: {
    name: 'Basic'
  }
  properties: {
    adminUserEnabled: true
  }
}
```

Lets deploy the Azure Container Registry first using the bicep code:
```
1. change the name of the az container registry in containerregistry.bicep
```

We need to login to Azure so we can issue the next commands.
```
2. az login 
```

Now create a resource group (change region/name as desired)
```
3. az group create -l westeurope -n iscachefunction
```

The next step deploys the bicep file to the iscachefunction resource group
```
4. az deployment group create --resource-group iscachefunction --template-file .\containerregistry.bicep
```

This step logs us in to this container registry so we can upload our docker image to it.
```
5. az acr login --name NAMEOFYOURCONTAINERREGISTRY
```

Now we will link our local docker image to a docker image hosted in the container registry. 
```
6. docker tag myacc/azurefunctionsimage:v0.0.2 cachetwofunctionacr.azurecr.io/azurefunctionsimage:v0.0.2
```

And finally we will upload our image.
```
7. docker push cachetwofunctionacr.azurecr.io/azurefunctionsimage:v0.0.2
```

#### Creating the Azure Function
The following Bicep snippet lists the resources required to run the Azure function.

```
resource appServicePlan 'Microsoft.Web/serverFarms@2020-06-01' = {
  name: 'cachefunctionappplan'
  location: 'westeurope'
  sku: {
    name: 'EP1'
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

resource storageAccount 'Microsoft.Storage/storageAccounts@2019-06-01' = {
  name: 'cachefunctionstorage'
  location: 'westeurope'
  kind: 'StorageV2'
  sku: {
    name: 'Standard_LRS'
    tier: 'Standard'
  }
}

resource azureFunction 'Microsoft.Web/sites@2020-12-01' = {
  name: 'iscachefunction'
  location: 'westeurope'
  kind: 'functionapp'
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'DOCKER|cachetwofunctionacr.azurecr.io/azurefunctionsimage:v0.0.2'               
      appSettings: [
        {
          name: 'AzureWebJobsStorage'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${environment().suffixes.storage};AccountKey=${listKeys(storageAccount.id, storageAccount.apiVersion).keys[0].value}'
        }
        {
          name: 'WEBSITE_CONTENTAZUREFILECONNECTIONSTRING'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${environment().suffixes.storage};AccountKey=${listKeys(storageAccount.id, storageAccount.apiVersion).keys[0].value}'
        }
        {
          name: 'WEBSITES_ENABLE_APP_SERVICE_STORAGE'
          value: 'false'
        }
        {
          name: 'DOCKER_REGISTRY_SERVER_URL'
          value: 'cachetwofunctionacr.azurecr.io'
        }
        {
          name: 'DOCKER_REGISTRY_SERVER_USERNAME'
          value: 'cachetwofunctionacr'
        }
        {
          name: 'DOCKER_REGISTRY_SERVER_PASSWORD'
          value: 'XEbEMIxsEZL4gSyU/YnkHUbfYY######'
        }
      ]
    }
  }
}

resource ipAddress 'Microsoft.Network/publicIPAddresses@2021-05-01' = {
  name: 'cachefunctionpublicip'
  location: 'westeurope'
  sku: {
    name: 'Standard'
  }
  properties: {
    publicIPAllocationMethod: 'Static'
  }
}

resource NatGateway 'Microsoft.Network/natGateways@2021-05-01' = {
  name: 'natgatewayforstaticip'
  location: 'westeurope'
  properties: {
    publicIpAddresses: [
      {
        id: ipAddress.id
      }
    ]
  }
  sku: {
    name: 'Standard'
  }
}
```

In the previous step we uploaded our container image to the container registry. We will need to configure our Azure Function to run this container, and get access to our ACR.

1. go to the Azure portal and copy the username password from the Azure Container Registry.
2. update the docker values in the infrastructure bicep file, and change the linuxFxVersion to link to your own azure container registry and your specific docker image.
```
        {
          name: 'DOCKER_REGISTRY_SERVER_URL'
          value: 'cachetwofunctionacr.azurecr.io'
        }
        {
          name: 'DOCKER_REGISTRY_SERVER_USERNAME'
          value: 'cachetwofunctionacr'
        }
        {
          name: 'DOCKER_REGISTRY_SERVER_PASSWORD'
          value: 'XEbEMIxsEZL4gSyU/YnkHUbfYY######'
        }
```

Now finally, lets deploy the rest of the infrastructure to Azure.
```
3. az deployment group create --resource-group yourrgname --template-file .\infrastructure.bicep
```
At this point everything should be working. If it's not, check the logs by going to https://{replacewithyourfunctionname}.scm.azurewebsites.net/api/logs/docker , there you might see what went wrong. Maybe you forgot to change the docker registry server username & password? 

I hope you found this useful! 

 


