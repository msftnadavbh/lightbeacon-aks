# Running AKS the right way - LightSpeed project on AKS

# Intro
Using Azure Kubernetes Service can be challenging,
Building it the right way can be difficult and best practices can sometimes be confusing.
In this guide, I'm going to explain how to build and deploy AKS the right way for demanding workloads using __Ephemeral OS__ and __nodepools__.
This blueprint will create a good base of using AKS and utilzing best practices to maximize performance.


## Prerequesits

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

Create the Resource Group :

`az group create -n testaks-rg -l eastus`


After creation has been finished, as we're using this for a Development environment.
I'm using __East US__ for demonstration purposes but you can use w/e region close to you.
I'm going to use version 1.19 with 2 nodepools :

`az aks create -n aks-eus -g rg1-eus -k 1.19.0 -c 2`

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

Now let's verify that our replica is up and running :

`az acr replication show -n centralus -r youracrname -o table`

__Status__ should be __Ready__. If it's still syncing, give it a few minutes and then try again.

We will use a simple Dockerfile that runs a PHP code that will throw an IP out. That's it.
It's included in the __source__ folder available here.
Let's navigate to the __source__ folder, and use __acr__ __build__ command to build the Docker Image and to push it to our registry :

`az acr build -t sample/dockertest:{{.Run.ID}} -r youracrname .`

This should create a sample repositry under your __Azure Container Registry__ created in the first step.
As we have Geo Replication enabled, Azure will automatically replicate the image to the secondary region.

Once the build is done, let's attach our ACR to our 2 AKS clusters to make life easier :

`az aks update -n aks-eus -g rg1-eus --attach-acr youracrname`

Second one :

`az aks update -n aks-cus -g rg2-cus --attach-acr youracrname`

Great. Now we have 2 AKS clusters on 2 different regions, each connected to the right replica,
With a test Docker Image available on them.

