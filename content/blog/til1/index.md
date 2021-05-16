---
title: Today I learned...
date: "2021-05-16T12:00:00.000Z"
description: "A place for my daily learnings"
---

This will become the first in (hopefully) a long running series of blogs, so let me explain:

## Why am I doing this?

A lot of my previous blogs have been reliant on me fully completing something. Whilst I want to continue to do this, there are so many small learnings and pieces of DevOps that I learn about that have either helped me solve a problem or have been a new discovery for me. I'm also very aware that I have more to learn, so [learning in public](https://www.swyx.io/learn-in-public/) with the opportunity of people being able to reach out to me to improve my mental model of a topic if I've misunderstood something sounds great! 

## What do I expect from this?

Best-case, I help someone solve a problem. Worst-case, I have a place to refer to if a problem arises again or I can just look back on these blogs later on in my career.

## Packer Variables in Azure Pipelines

I had a colleague who was trying to get Packer running in a pipeline to build an image in Azure. The pipeline was failing with the following error:

```
Failed to interpolate "client_secret": "{{user `AZURE_CLIENT_SECRET`}}"; error:
template: root:1:2: executing "root" at <user `AZURE_CLIENT_SECRET`>: error
calling user: Error: variable not set: AZURE_CLIENT_SECRET: Please make sure
that the variable you're referencing has been defined; Packer treats all
variables used to interpolate other user variables as required.
```

When I first looked at the repo, This was the JSON (Truncated):

```
{
  "variables": {
    "client_id": "{{user `AZURE_CLIENT_ID`}}",
    "client_secret": "{{user `AZURE_CLIENT_SECRET`}}"
  },
  "builders": [
      {
      "type": "azure-arm",
      "client_id": "{{`AZURE_CLIENT_ID`}}",
      "client_secret": "{{`AZURE_CLIENT_SECRET`}}"
      }
  ]
}
```
Having rarely used Packer before, I saw that the error was probably relating to the ```user``` in the variables and that maybe it shouldn't/is not allowed to be there. I search around and see that we want to export an environment variable (AZURE\_CLIENT\_ID) and set it to client\_id. This is what [env](https://www.packer.io/docs/templates/hcl_templates/functions/contextual/env) does! So I replaced ```user``` with ```env``` and got a new error:

```
{"error":"unauthorized_client","error_description":"AADSTS700016: Application with identifier 'AZURE_CLIENT_ID' was not found in the directory '***'}
```

My initial thoughts were "Ahhh I've seen something like this before, it must be the service principal permissions right? I checked the subscription and sure enough, the principal was there with the permissions it needed. Some more digging online revealed [this blog](https://compositecode.blog/2019/08/06/error-build-azure-arm-errored-adal-failed-to-execute-the-refresh-request-error/), which notes perfectly the issue:

```
What this does is take the environment variables and passes them to what will be user variables for the builder block to make use of.
```

Cool, so the variables you set at the top of a Packer file become user variables(?) and must be referenced that way! So the pipeline was interpreting ```"{{`AZURE_CLIENT_ID`}}"``` as a literal string! To fix this, I changed the builder block above to the following:

```
"builders": [
    {
    "type": "azure-arm",
    "client_id": "{{user `client_id`}}",
    "client_secret": "{{user `client_secret`}}"
    }
]
```

and it worked!
