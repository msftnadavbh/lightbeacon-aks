# LightBeacon project - Running AKS the right way 
## Intro
Using Azure Kubernetes Service can be challenging,
Building it the right way can be difficult and best practices can sometimes be confusing.
In this guide, I'm going to explain how to build and deploy AKS demanding workloads using __Ephemeral OS disks__ and __nodepools__.
This blueprint will create a good base of using AKS and utilizing best practices to maximize performance.
I have created LightBeacon in order to make AKS accessible for people who're new to AKS as a whole and wish to have a guide that is simplified and accessible.

__Important Note__ : I will not be addressing Security and/or Identity governance in this guide. 


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

## Reconfiguring AKS 

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

Running our `az aks nodepool list --cluster-name testaks --resource-group testaks-rg -o table` command will show 3 nodepools now :


![result](/images/6.png)

Now, before we eliminate our old nodepool, let's verify a few things.
Let's use __kubectl get podes -n kube-system__ to view our systempods, 
And deep dive into one of them using __kubectl describe pod__.

For this example, let's checkout our __CoreDNS__ pod and scroll down to __Tolerations__:

![result](/images/7.png)

Remember our __CriticalAddonsOnly__ taint? Azure automatically tags systempods this way.
In this method, we'll make sure that after our cluster has lost it's original nodepool, it will keep running, using our new __Ephemeral System nodepool__.
Let's delete our original nodepool :

` az aks nodepool delete --cluster-name testaks --resource-group testaks-rg --name nodepool1 `

Azure will now gracefully cordon & drain all of the nodes in the original nodepool and delete them afterwards.
After the deletion has been completed, let's use the az aks nodepool list command again to view our state :

![result](/images/8.png)

Now we have 2 nodepools. One marked as __System__, and one marked as __User__.

Let's check what's going on with our system pods :

![result](/images/9.png)

Great, seems like everything is running just fine.

Now, to shake things a bit, let's delete one of the kube-proxy pods to see what's going to happen :

![result](/images/10.png)

Great, now, let's run `kubectl get pods -n kube-system` again to check out our new pod :

![result](/images/11.png)

Let's see where was it scheduled to, using `kubectl describe pod [podname] -n kube-system` :

![result](/images/12.png)

The pod was successfully rescheduled to our __System__ nodepool, which means we have configured our cluster correctly.

What will happen if we'll schedule a normal non-system workload to our cluster?
Let's find out.

For this, let's create an Azure Container Registry that will host our test application.

`az acr create -n youracrname -g testaks-rg --sku Standard`

Now, let's attach our newly created Container Registry to AKS, the easy way :

`az aks update -n testaks -g testaks-rg --attach-acr youracrname`



