---
title: ClamAV API in Azure
date: "2021-01-06T08:35:00.000Z"
description: "Read on to find out how to deploy ClamAV as an API onto an Azure Web App"
---

ClamAV is an open-source antivirus solution that uses a virus database to detect and remove malware. One of the largest issues with ClamAV is that it's protocol (Clamd) contains command such as shutdown, so exposing clamd directly to external services is not a great idea.

## Enter REST

To prevent the clamd protocol being directly exposed, I decided to use a [REST Proxy](https://github.com/solita/clamav-rest) (there are a few of these that exist in different languages). The REST service exposes the clamd status path (```/```) as well as the scan result and scan reply path (```scan``` and ```scanReply```). 

Given how tightly coupled the REST service and the ClamAV server are, I decided that a docker-compose file would be best suited for this task.

The solution used is similar to [this](https://blog.theodo.com/2017/11/implement-antivirus-api-10-min/), with changes so that the docker-compose file looks like:

```
version: '3.3'

services: 
  clamav-rest: 
    image: lokori/clamav-rest
    ports: 
      - "80:8080"
    links: 
      - clamav-server
    environment: 
      CLAMD_HOST: clamav-server
    restart: always


  clamav-server: 
    image: mkodockx/docker-clamav
```

## How to get it working in Azure

Now that I had the docker-compose file working, I needed a way to get this running in Azure. Thankfully, Azure have a feature that allows you to create a multi-container app within a webapp. You can read more on this [here](https://docs.microsoft.com/en-us/azure/app-service/tutorial-multi-container-app#create-a-docker-compose-app).

At first, I followed this tutorial through an authenticated terminal. However, the end result never worked and it appeared as if the docker-compose file was corrupted in the portal. Sure enough, the following command works *in Cloudshell only*:

```
az webapp create --resource-group myResourceGroup --plan myAppServicePlan --name <app-name> --multicontainer-config-type compose --multicontainer-config-file docker-compose.yml
```

and thats it!

Not only do you prevent the clamd service from being exposed, but you also allow for an easier method of network access restrictions and a solution that can act as a central utility for all the applications on your Azure tenant.

You can test the REST API through curl like so:

```
echo "hi" > test.txt
curl -F "name=blabla" -F "file=@./test.txt" https://<your app service domain>/scan

Everything ok : true
```

and to test a vulnerability, you can use an [eicar file](https://www.eicar.org/?page_id=3950) by placing the following in a file:

```
X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*
```

and you can see the result as

```
curl -F "name=blabla" -F "file=@./eicar.txt" https://<your app service domain>/scan

Everything ok : false
```
