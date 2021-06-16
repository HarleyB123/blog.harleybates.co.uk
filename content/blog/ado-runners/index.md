---
title: Azure DevOps Runners with Kubernetes and KEDA
date: "2021-06-16T12:00:00.000Z"
description: "Learn how I deployed Azure DevOps Self-Hosted Agents to Kubernetes and scaled based on queue size"
---

Azure DevOps provides the option to deploy self-hosted agents using Docker [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops). This is really useful, as sometimes I come across the issue where I've had to allowlist an agents IP to a resource in Azure in the pipeline, only for the change to have not occcured in time for the next step and my pipeline to fail!

Having a static IP that is known solves all of these problems 🙌

## Steps to deploy

The Azure Documentation to running self-hosted agents is easy enough to get started with and provides a YAML Deployment template for running the agents inside of a cluster easily! Since the solution is being deployed in two different Azure DevOps locations (thus meaning different URL's and PAT tokens), as well as integrations that are mentioned later on in this blog, Helm felt like the right tool for the job - as I am able to create a GitHub workflow that can upgrade either (or both!) of
the ADO environments independently from each other and pass through variables via the ```values.yaml``` file or using the ```--set``` flag in the Helm upgrade cli command. The main part of my deployment looks like this:

```yaml
containers:
  - name: "ado-runner-container"
    securityContext:
      {{- toYaml .Values.securityContext | nindent 12 }}
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
    imagePullPolicy: {{ .Values.image.pullPolicy }}
    env:
      - name: AZP_URL
        value: {{ .Values.AZPURL }}
      - name: AZP_POOL
        value: {{ .Values.AZPPOOL }}
      - name: AZP_TOKEN
        valueFrom:
          secretKeyRef:
            name: adopat
            key: token 
```

## Managing secrets

Despite the Azure DevOps agent requiring very few values to run, one of those is a PAT token (which I recommend is one from a service account user created in Azure DevOps!). There are many ways to pass through secrets to Kubernetes, including Azure's own recommmendation of creating a generic secret on the AKS cluster itself!

Ideally, I wanted a solution where the secrets are either stored or referenced outside of the cluster, so that I am able to change the values easily when it is time to rotate the tokens. This lead me to two options, the [CSI driver](https://docs.microsoft.com/en-us/azure/key-vault/general/key-vault-integrate-kubernetes) or [akv2k8s](https://akv2k8s.io/). The former looked a bit confusing to get setup, so I opted for akv2k8s!

This solution was really easy to implement and meant adding a dependency to the ```Chart.yaml``` file. With akv2k8s, you can either inject a secret or simply sync with key vault. The latter simply grabs the secret from key vault and then creates a secret in the cluster. 

The one issue with this tool that my colleague noted is that if a secret changes, the pods will still reference the old secret until such time as you kill the pods. This isn't the end of the world though (in fact - it's quite a good thing! You wouldn't want your app going down straight after a secret change!), a rolling update of some kind that triggers on the secret change could solve this issue.

The ```secrets.yaml``` file looks like this:

```yaml
apiVersion: spv.no/v2beta1
kind: AzureKeyVaultSecret
metadata:
  name: kls-secret-sync
  namespace: {{ .Values.NAMESPACE }}
spec:
  vault:
    name: youradorunnerskv # name of key vault
    object:
      name: {{ .Values.KVSECRETNAME }} # name of the akv object
      type: secret # akv object type
  output:
    secret:
      name: adopat
      dataKey: token
```

## Autoscaling with KEDA

One of the hardest features to implement when using agent runners is when to scale, as metrics such as CPU and Memory don't tell the full story. When looking at a tool to solve the issue of scaling a deployment based on an external metric, the only tool that fit the bill (and perfectly I might add), was [KEDA](https://keda.sh/) with their [Azure Pipelines Scaler](https://keda.sh/docs/2.3/scalers/azure-pipelines/).

KEDA uses the number of agents that are currently running a job to scale. This means that if I have a minimum of 5 agents (and all of them are currently in use!), then a user won't have to wait for one of those agents to become available (they will, however, have to wait for the agent to spin up - but this is around 15 seconds).

Overall, this was really easy to get setup, though I did have an issue where I was including the trailing slash on the end of the URL for the Azure DevOps Organisation, which took a while to debug and solve by looking at the code for KEDA. 

I also don't like how you have to get the Azure DevOps Agent Pool ID (a numerical value rather than the name that you can retrieve using [this](https://docs.microsoft.com/en-us/cli/azure/pipelines/pool?view=azure-cli-latest#az_pipelines_pool_list) command), but thankfully it's static once you've retrieved the value. I also had issues in passing this value from ```values.yaml```, so I discovered Helm's [string conversion](https://helm.sh/docs/chart_best_practices/values/#make-types-clear) to
resolve this.

The ```keda.yaml``` file looks like this:

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
    name: pipeline-trigger-auth
spec:
  secretTargetRef:
  - parameter: personalAccessToken
    name: adopat
    key: token
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-pipelines-scaledobject
spec:
  scaleTargetRef:
    name: ado-runners
  minReplicaCount: 5
  maxReplicaCount: 20
  triggers:
  - type: azure-pipelines
    metadata:
      poolID: !!string {{ .Values.POOLID }}
      organizationURLFromEnv: "AZP_URL"
    authenticationRef:
      name: pipeline-trigger-auth
```

## Docker in Docker

One of the interesting elements that I noted on the Dockerfile provided by Azure was the mounting of the Docker socket (thus enabling root access through the daemon). After talking to someone at Azure during a hackathon, I was told that it was there to run Docker Pipelines inside the container - with daemonless options like Sysbox being the alternative. 

However, I noticed that you actually get an error when trying to run a Docker pipeline task inside an agent that is running on a container:

```
##[error]Container feature is not supported when agent is already running inside container. Please reference documentation (https://go.microsoft.com/fwlink/?linkid=875268)
```

You can find the discussion on this topic [here](https://github.com/MicrosoftDocs/azure-devops-docs/issues/10621) and [here](https://github.com/microsoft/azure-pipelines-agent/pull/1619).

In short, I don't think it's possible to run Docker pipeline tasks when using self-hosted agents that also run in a Docker container.

## Future

The solution mentioned above is not just for me or my team, but for multiple cloud teams. The issue around charging for this is that a cloud team should not be charged if they do not use the self-hosted agents.

For this, A colleague mentioned a method where you could create a namespace in the cluster for each cloud team (an agent pool is also created for each team in the Azure DevOps Organisation) which scales the agents down to 0. From this, a tool called [Kubecost](https://www.kubecost.com/) is being considered so that I can not only 'rightsize' the cluster, but also see the cost for each namespace - thus identifying how much each team should be charged.
