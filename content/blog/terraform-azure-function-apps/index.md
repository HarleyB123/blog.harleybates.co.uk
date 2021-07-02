---
title: Debugging the Azure Terraform Provider
date: "2021-07-02T08:33:00.000Z"
description: "Read on to find out how Terraform caused an incident, how I identified the issue and future solutions"
---

Within my workplace, Terraform is used frequently to deploy infrastructure in Azure. On the whole, the Azure Terraform Provider is clear on what it will change, or descriptive enough in the errors for you to figure out what's not allowed from an Azure API perspective. However, recently an incident with a function app terraform deployment caused an incident - here was how I debugged it.

## The incident

As part of a production change, a function app was update to add the correct VNET name in. The plan only showed this change and nothing else. The Terraform applied, and the function app started failing.

In my team, Terraform does not manage any of the application settings. We use a custom tool that allows for the developers to manage application settings in JSON files within source code, so that changes to app configuration can be made separately to the infrastructure.

After a while debugging and reviewing the activity logs, the team discovered that ```WEBSITE_CONTENTAZUREFILECONNECTIONSTRING``` setting was removed entirely! Despite no indication of this happening in the plan!

## Debugging

The first thoughts from the team were around adding ```ignore_changes``` blocks to the application setting, but this doesn't explain why the setting changed in the first place, and why Terraform didn't identify it in the plan! From previous experience contributing to the Azure Terraform Provider - I decided to have a look at the code for the Azure Function App resource and see if I could find anything from that side.

[Here](https://github.com/terraform-providers/terraform-provider-azurerm/blob/master/azurerm/internal/services/web/function_app_resource.go) is the code for creating a function app in Azure through Terraform. If you look at [this line](https://github.com/terraform-providers/terraform-provider-azurerm/blob/master/azurerm/internal/services/web/function_app_resource.go#L308), which can also be seen in the update function
[here](https://github.com/terraform-providers/terraform-provider-azurerm/blob/master/azurerm/internal/services/web/function_app_resource.go#L428), Terraform is calling a function - ```getBasicFunctionAppAppSettings```. 

Sure enough, when you follow that function to [here](https://github.com/terraform-providers/terraform-provider-azurerm/blob/3add6b0319d9121bb329187072ab6256fbcb69c2/azurerm/internal/services/web/function_app.go#L221), you can see that behind the scenes, Azure is setting these environment variables! In fact, when looking back at the activity logs for the incident, I noted that ```WEBSITE_CONTENTSHARE``` was also removed from the application settings!

### Why is Azure doing this?

From a following of discussions, there was no API in Azure to create the app function, and so [setting internal environment variables](https://github.com/Azure/azure-rest-api-specs/issues/3750) is how you get around it. 

This has since been fixed from discussions [here](https://github.com/Azure/azure-sdk-for-go/issues/2397) and [here](https://github.com/Azure/azure-functions-host/issues/3994), but the azure provider is still using the older API version and this logic is considered a breaking change, so will be [removed in version 3](https://github.com/terraform-providers/terraform-provider-azurerm/blob/3add6b0319d9121bb329187072ab6256fbcb69c2/azurerm/internal/services/web/function_app.go#L231) of the AzureRM Terraform provider, see a PR for it [here](https://github.com/terraform-providers/terraform-provider-azurerm/pull/10494).

## Others feeling the pain

Whilst tough to find, others are feeling the pain of this hidden alteration of environment variables (shock - I know!), with [here](https://github.com/terraform-providers/terraform-provider-azurerm/issues/10499) noting the exact same concern:

> This is very dangerous because it can cause an "hidden swap" of the running software from a stable version to an unstable one.

The PR to solve this problem also raises a comment similar to this [here](https://github.com/terraform-providers/terraform-provider-azurerm/pull/10494#issuecomment-830756983):

> Forcing the creation of that setting inside the Create and Update with fixed values breaks the behavior in functions with slots when swapping. Which is very damaging and risky. It happened to me in production twice before understanding there was a bug.

## Temporary Solution?

In general, there isn't one that doesn't restrict Terraform entirely, see [here](https://github.com/terraform-providers/terraform-provider-azurerm/pull/10494#issuecomment-854515519).

For my team, the solution will be to either:

- Run the custom tooling for application settings after every deployment
- Edit the custom tooling so it pulls down the existing values for ```WEBSITE_CONTENTAZUREFILECONNECTIONSTRING``` and ```WEBSITE_CONTENTSHARE``` and appending them to the configuration specified in the JSON files in source code!

Overall this was quite an interesting upstream bug to encounter - and certainly one that should be flagged to the wider Terraform Azure community!
