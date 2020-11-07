---
title: Reporting on Azure Resources Programmatically
date: "2020-11-07T12:00:00.000Z"
description: "Gathering a holistic view on the configuration of a resource type in Azure is tough - here's how I managed to do it through Python!"
---

Let's say you want to find out whether all your app services have HTTPS Only, or the latest TLS Version - How would you do that? Once found out, how would you track that over time?

In a small enough Azure tenant, you could easily go through each resource. Alternatively, a well structured tenant may have policy in place to enforce HTTPS Only and TLS 1.2. Perhaps all of your infrastructure is in IaC - and you can query this information through there.

Since none of these were possible in my scenario - I created a Python tool that can scan each webapp for you and put the information in a log aggregator for future querying!
 
Firstly - we would need to list out all of the subscriptions and resource groups, like so:

```
from azure.common.credentials import ServicePrincipalCredentials
from azure.mgmt.resource import ResourceManagementClient
from azure.mgmt.subscription import SubscriptionClient

client_id = "" 
secret = "" 
TENANT = ""

CREDENTIALS = ServicePrincipalCredentials(
    client_id=client_id, secret=secret, tenant=TENANT,
)


def list_subscriptions():
    client = SubscriptionClient(CREDENTIALS)
    # ignore disabled subscriptions
    subs = [
        sub.subscription_id
        for sub in client.subscriptions.list()
        if sub.state.value == "Enabled"
    ]

    return subs


def list_resource_groups():
    subs = list_subscriptions()
    resource_groups = {}

    for sub in subs:
        resource_group_client = ResourceManagementClient(CREDENTIALS, sub)
        rgs = resource_group_client.resource_groups.list()

        # generate a list of resource groups
        groups = [rg.name for rg in rgs]

        # create a nested dictionary -- {"sub_id": {[rg1, rg2, rg3]}, "sub_id2": {[rg1, rg2, rg3]}}
        resource_groups[sub] = groups
    return resource_groups


def main():
    rgs = list_resource_groups()


if __name__ == "__main__":
    main()
```
Then I want to gather all the webapps and some import data about them! Since this will eventually be going into a log aggregator, I want to use ```json.dumps()``` so that it is valid JSON later on. Interestingly, I wanted Diagnostic Logs - which required me to pass through the name of the Diagnostic Log that I wanted to get. This means that in order for the script to provide a correct response - all app services need to have a diagnostic log with the same name (a future blog on how to do this is coming soon!).

```
import json

from azure.mgmt.monitor import MonitorManagementClient
from azure.mgmt.web import WebSiteManagementClient
from msrest.exceptions import ClientException


def get_all_webapps(credentials, rgs):
    all_webapps = []
    for sub, groups in rgs.items():
        web_client = WebSiteManagementClient(credentials, sub)
        monitor_client = MonitorManagementClient(credentials, sub)
        for rg in groups:
            for site in web_client.web_apps.list_by_resource_group(rg):
                get_config = web_client.web_apps.get_configuration(rg, site.name)
                resource_id = f"/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Web/sites/{site.name}"
                AuditLogs = "No Audit Logs"
                try:
                    monitor_client.diagnostic_settings.get(
                        resource_id, "diagnostic-log-name" # Place the name of your diagnostic log here!
                    )
                    AuditLogs = True
                except ClientException as ex:
                    pass

                webapp_data = dict(
                    {
                        "Subscription": sub,
                        "Resource Group": rg,
                        "App Service Name": site.name,
                        "HTTPS_ONLY": site.https_only,
                        "FTPS": get_config.ftps_state,
                        "TLS": get_config.min_tls_version,
                        "Always-On": get_config.always_on,
                        "Audit Logs": AuditLogs,
                        "Kind": site.kind,
                        "Location": site.location,
                    }
                )
                all_webapps.append(webapp_data)
    return json.dumps(all_webapps)
```

The final part of the script is to push this information to a log aggregator. Azure have already got the sample code for this [here](https://docs.microsoft.com/en-us/azure/azure-monitor/platform/data-collector-api#python-3-sample) - which I only had to modify slightly to fit my needs:

```
import base64
import datetime
import hashlib
import hmac
import json

import requests

# Update the customer ID to your Log Analytics workspace ID
customer_id = ""

# For the shared key, use either the primary or the secondary Connected Sources client authentication key
shared_key = ""

# The log type is the name of the event that is being submitted

#####################
######Functions######
#####################

# Build the API signature
def build_signature(
    customer_id, shared_key, date, content_length, method, content_type, resource
):
    x_headers = "x-ms-date:" + date
    string_to_hash = (
        method
        + "\n"
        + str(content_length)
        + "\n"
        + content_type
        + "\n"
        + x_headers
        + "\n"
        + resource
    )
    bytes_to_hash = bytes(string_to_hash, encoding="utf-8")
    decoded_key = base64.b64decode(shared_key)
    encoded_hash = base64.b64encode(
        hmac.new(decoded_key, bytes_to_hash, digestmod=hashlib.sha256).digest()
    ).decode()
    authorization = "SharedKey {}:{}".format(customer_id, encoded_hash)
    return authorization


# Build and send a request to the POST API
def post_data(customer_id, shared_key, body, log_type):
    method = "POST"
    content_type = "application/json"
    resource = "/api/logs"
    rfc1123date = datetime.datetime.utcnow().strftime("%a, %d %b %Y %H:%M:%S GMT")
    content_length = len(body)
    signature = build_signature(
        customer_id,
        shared_key,
        rfc1123date,
        content_length,
        method,
        content_type,
        resource,
    )
    uri = (
        "https://"
        + customer_id
        + ".ods.opinsights.azure.com"
        + resource
        + "?api-version=2016-04-01"
    )

    headers = {
        "content-type": content_type,
        "Authorization": signature,
        "Log-Type": log_type,
        "x-ms-date": rfc1123date,
    }

    response = requests.post(uri, data=body, headers=headers)
    if response.status_code >= 200 and response.status_code <= 299:
        print("Accepted")
    else:
        print("Response code: {}".format(response.status_code))


def post_to_log_aggregator(body, log_type):
    post_data(customer_id, shared_key, body, log_type)
```

And thats it! For simplicity and resuability over more resource types - I decided to make each one of these code snippets a seperate file. The first code snippet is called main.py, the second webapps.py and the final code snippet is called aggregation.py

You can see the code [here](https://github.com/HarleyB123/AzureWebAppReporting) for the end solution all pieced together!
