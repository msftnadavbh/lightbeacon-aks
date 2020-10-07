# kubefed-aks
using kubefed to federate AKS clusters

# Intro
Using Azure Kubernetes Service can be challenging,
In a scenario where you need to faciliate using Kubernetes Federation.
In this guide, I'm going to explain how to leverage kubefed to successfully construct a Kubernetes Federation between 2 AKS clusters.

## Prerequesits

We are to create two Resource Groups on differnet regions.
For the sake of the example, I will use East US and Central US.

First, make sure your Azure CLI is properly configured.
You can download Azure CLI from [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).

Download __kubectl__ if you haven't already :

`az aks install-cli`

Make sure you're logged in and you have __aks-preview__ extension installed :

`az login`

Then :

`az extension add -n aks-preview`

This will make stuff easier later on.

## Setting up the environment

Create the Resource Groups :

`az group create -n rg1-eus -l eastus`

`az group create -n rg2-cus -l centralus`
