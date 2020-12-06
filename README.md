# LightBeacon project - Running AKS the right way 
## Intro
Using Azure Kubernetes Service can be challenging,
Building it the right way can be difficult and best practices can sometimes be confusing.
In this guide, I'm going to explain how to build and deploy AKS the right way for demanding workloads using __Ephemeral OS disks__ and __nodepools__.
This blueprint will create a good base of using AKS and utilizing best practices to maximize performance.
I have created LightBeacon in order to make AKS accessible for people who're new to AKS as a whole and wish to have a guide that is simplified and accessible.


## Prerequesits

* An Azure subscription

First, make sure your Azure CLI is properly configured on your workstation.
You can download Azure CLI from [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).

Download __kubectl__ if you haven't already :

`sudo az aks install-cli`

Make sure you're logged in and you have __aks-preview__ extension installed :

`az login`

Then :

`az extension add -n aks-preview`

This will make stuff easier later on.

Throughout the tutorial I'll be using WSL 2 with Ubuntu 20 on Windows 10.

## Setting up the environment

Create the Resource Group :

`az group create -n testaks-rg -l eastus`


I'm using __East US__ for demonstration purposes but you can use w/e region close to you.
I'm going to use version 1.19.3 as I'm using this for a Dev environment.
We'll build the first nodepool normally and then we will make adjustments.

Let's use this command to create the initial cluster :

`az aks create --name testaks --resource-group testaks-rg --kubernetes-version 1.19.3 --node-count 3  --node-vm-size Standard_D2_v3 --network-plugin azure --generate-ssh-keys --debug `


__Note__: ARM will automatically select the region where the resource group is located at, therefore there is no need for an __-l__ trigger unless you want to specify a different region.

Alright, now we have the cluster available in __East US__ :

![AKS overview](/images/1.png)

Now, let's connect to the fresh cluster.

 Let's use __get-credentials__ to connect to our cluster :

`az aks get-credentials -n testaks -g testaks-rg`

Now, we can run __kubectl get nodes -o wide__ to see what we have currently :

![result](/images/2.png)

Awesome, we have 3 nodes running in a healthy state.

Let's view the current state of our nodepool :

`az aks nodepool list --cluster-name testaks --resource-group testaks-rg -o table`

![result](/images/3.png)

Great, we have one nodepool classified as __System__.

__System__ is the default nodepool selected at the initial creation of the cluster.
This is to make sure that all __kube-system__ related pods can be deployed without any issue so the cluster's creation can be completed successfully.

We can verify that using the below command :

![result](/images/4.png)

In the current state, our single nodepool will accept all deployments and will co-host the system pods with the application pods.
Now, let's change the overall structure of the cluster, to accommodate 2 more nodepools, with a tweak.
We'll add another __System__ nodepool that will host our system pods using __Ephemeral__ disks which will be much faster,
and a __User__ nodepool that will host our application and will be dedicated to it.

Using this structure, we're omitting issues like __IOPS__ and __Networking throughput__ that can affect our application and our system pods/services,
by sepearting them from eachother and implementing better governance into AKS.

Let's add our new System nodepool.
Pay close attention to __--node-taints__ trigger added in the command and to __--node-osdisk-type__ trigger :

`az aks nodepool add --name ephsystem --cluster-name testaks --resource-group testaks-rg --node-vm-size Standard_DS3_v2 --node-osdisk-type Ephemeral --node-taints CriticalAddonsOnly=true:NoSchedule --node-count 2 --mode System`

We've told Azure to create another System nodepool,
Using a different VM size to accomodate Ephemeral OS disk because of disk cache considerations,
We've also added a special __taint__ to stop non-system workloads from being scheduled to this new nodepool.

After our command has finished running, let's use `az aks nodepool list --cluster-name testaks --resource-group testaks-rg -o table` to check our nodepool state :

![result](/images/5.png)

Great, our new System nodepool is up and running.

Now, let's create the __User__ nodepool for our application :

  `az aks nodepool add --name appnopool --cluster-name testaks --resource-group testaks-rg --node-vm-size Standard_D2_v3 --node-count 2 --mode User`

__Note__ : Pay attention that we're back to the original VM size since we do not have a special need for Ephemeral disks. also,
I'm using a small VM size since this is a dev environment, in a production workload, use the right VM size that is applicable to your application.
Also, there are no taints here since we want this nodepool to be accessible.




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

