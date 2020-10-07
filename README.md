# kubefed-aks
using kubefed to federate AKS clusters

# Intro
Using Azure Kubernetes Service can be challenging,
In a scenario where you need to faciliate using Kubernetes Federation.
In this guide, I'm going to explain how to leverage kubefed to successfully construct a Kubernetes Federation between 2 AKS clusters.

## Prerequesits

We will create 2 Resource Groups on 2 differnet regions.
For the sake of the example, I will use East US and Central US.

First, make sure your Azure CLI is properly configured on your workstation.
You can download Azure CLI from [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).

Download __kubectl__ if you haven't already :

`az aks install-cli`

Make sure you're logged in and you have __aks-preview__ extension installed :

`az login`

Then :

`az extension add -n aks-preview`

This will make stuff easier later on.

Throughout the tutorial I'll be using WSL 2 on Windows 10.

## Setting up the environment

Create the Resource Groups :

`az group create -n rg1-eus -l eastus`

`az group create -n rg2-cus -l centralus`

After creation has been finished, as we're using this for a Development environment,
I'm going to use version 1.19 on both of the clusters with 2 nodes each :

`az aks create -n aks-eus -g rg1-eus -k 1.19.0 -c 2`

Then for the Central US cluster :

`az aks create -n aks-cus -g rg2-cus -k 1.19.0 -c 2`

__Note__: ARM will automatically select the region where the resource group is located at, therefore there is no need for an __-l__ trigger unless you want to specify a different region.

Alright, now we have 2 identical clusters in differnet regions -

![2 AKS clusters](/images/1.png)

I'm going to use a geo replicated Azure Container Registry for the sake of this tutorial.
To create one, use this :

`az acr create -n youracrname -g rg1-eus --sku Premium`

__Note:__ The Premium SKU is needed for the Geo Replication feature.

After it's finished creating, we will set up a replica for Central US :

`az acr replication create -r youracrname -l centralus`

Since our registry is empty, this should be pretty fast.
You'll get an output saying that your replica has been created and that it's status has changed to __Syncing__:

![syncing replica](/images/2.png)
