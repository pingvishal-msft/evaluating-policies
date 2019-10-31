---
title: Authoring Evaluation Policies
description: Ensure artifacts adhere to custom policies
ms.topic: reference
ms.prod: devops
ms.technology: devops-cicd
ms.manager: shashban
ms.author: shashban
author: shashankbansal
ms.date: 10/31/2019
monikerRange: 'azure-devops'
---

# Authoring Artifact Evaluation Policies

Adding a Check to Evaluate Artifact requires the custom policy to be configured. This guide describes how custom policies can be created.

> [!NOTE]
> Currently, this Check works with container image artifacts only

## Pre-requisites

Familiarize yourself with [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/) query language. Basics will do.

## Creating custom policies

Below are the sample policies shared. Based on your requirements, you can build your own set of policies.

### Check specific project/pipeline 

This policy checks if the images are built by Azure Pipelines and Pipeline-foo. For this to work, the pipeline definition should override the name field to something like: AzureDevOps_$(BuildDefinitionName)_$(Date:yyyyMMdd)$(Rev:.r)see more about naming pipeline runs [here] (https://docs.microsoft.com/en-us/azure/devops/pipelines/process/run-number?view=azure-devops&tabs=yaml)

```
allowedBuilder := "AzureDevOps_pipeline-foo"

checkBuilder[errors] {
	trace("Check if images are built by Azure Pipelines")
	resourceUri := values[index].build.resourceUri	
	image := fetchImage(resourceUri)
	builder := values[index].build.build.provenance.builderVersion
	trace(sprintf("%s: builder", [builder]))
	not startswith(builder, allowedBuilder)
	errors := sprintf("%s: image not built by Azure Pipeline [%s]", [image,builder])
}

fetchRegistry(uri) = reg {
	out := regex.find_n("//.*/", uri, 1)
	reg = trim(out[0], "/")
}

fetchImage(uri) = img {
	out := regex.find_n("/.*@", uri, 1)
	img := trim(out[0], "/@")
}
```

### Check whitelisted registries

This policy checks if the images are from whitelisted registries only.

```
whitelist = {
 "gcr.io/myrepo",
 "raireg1.azurecr.io"
}

checkregistries[errors] {
	trace(sprintf("Whitelisted registries: %s", [concat(", ", whitelist)]))
	resourceUri := values[index].image.resourceUri
	registry := fetchRegistry(resourceUri)
	image := fetchImage(resourceUri)
	not whitelist[registry]
	errors := sprintf("%s: source registry not permitted", [image]) 
}

fetchRegistry(uri) = reg {
	out := regex.find_n("//.*/", uri, 1)
	reg = trim(out[0], "/")
}

fetchImage(uri) = img {
	out := regex.find_n("/.*@", uri, 1)
	img := trim(out[0], "/@")
}
```

### Check forbidden ports 

This policy checks for any forbidden ports exposed in the container image.

```
forbiddenPorts = {
	"80",
	"22"
}

checkExposedPorts[errors] {
	trace(sprintf("Checking for forbidden exposed ports: %s", [concat(", ", forbiddenPorts)]))
	layerInfos := values[index].image.image.layerInfo
	layerInfos[x].directive == "EXPOSE"
	resourceUri := values[index].image.resourceUri
	image := fetchImage(resourceUri)
	ports := layerInfos[x].arguments
	trace(sprintf("exposed ports: %s", [ports]))
	forbiddenPorts[ports]
	errors := sprintf("%s: image exposes forbidden port %s", [image,ports])
}

fetchRegistry(uri) = reg {
	out := regex.find_n("//.*/", uri, 1)
	reg = trim(out[0], "/")
}

fetchImage(uri) = img {
	out := regex.find_n("/.*@", uri, 1)
	img := trim(out[0], "/@")
}

```

### Check prior deployments

This policy checks if the image has been pre-deployed to one/more of the environments before being deployed to specific environment/resources with Check configured. 

```
predeployedEnvironments = {
	"env/resource1",
	"env2/resource3"
}

checkDeployedEnvironments[errors] {
	trace(sprintf("Checking if the image has been pre-deployed to one of: [%s]", [concat(", ", predeployedEnvironments)]))
	deployments := values[index].deployment
	deployedAddress := deployments[i].deployment.address
	trace(sprintf("deployed to : %s",[deployedAddress]))
	resourceUri := deployments[i].resourceUri
	image := fetchImage(resourceUri)
	not predeployedEnvironments[deployedAddress]
	trace(sprintf("%s: fails pre-deployed environment condition. found %s", [image,deployedAddress]))
	errors := sprintf("image %s fails pre-deployed environment condition. found %s", [image,deployedAddress])
}

fetchRegistry(uri) = reg {
	out := regex.find_n("//.*/", uri, 1)
	reg = trim(out[0], "/")
}

fetchImage(uri) = img {
	out := regex.find_n("/.*@", uri, 1)
	img := trim(out[0], "/@")
}
```
