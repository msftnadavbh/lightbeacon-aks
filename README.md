# LightBeacon project - Running Azure Kubernetes Service the right way 
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

__Note__ : I will be using the names 'testaks' for the cluster and 'testaks-rg' for the resource group. Change it to whatever you like.

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
Let's use __kubectl get pods -n kube-system__ to view our systempods, 
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

__Note__ : change 'youracrname' to a unique name for your Container Registry.

Now, let's attach our newly created Container Registry to AKS, the easy way :

`az aks update -n testaks -g testaks-rg --attach-acr youracrname`

We are good to go now.

Let's git clone this repo into a folder called __demo__ :

` mkdir demo && cd demo && git clone https://github.com/msftnadavbh/lightbeacon-aks.git && cd lightbeacon-aks/source`

We've have cloned the repo and we're into the __source__ folder, which holds a nice Dockerfile running a sample PHP webpage.
Let's use __az acr build__ to build the application directly into our ACR :

`az acr build -t sample/webpage -r youracrname .`

Now we're building the Docker image without the need of a Docker engine, directly on Azure.
If you have Docker installed, you can try to fetch it using :

`docker pull youracrname.azurecr.io/sample/webpage:latest`


Good, now you have a working Azure Container Registry which is attached to AKS with a test web application.
Now, let's try and deploy it - and see what happens.

Navigate back to __source__ folder, there you'll find a cool file called __deployment.yml__.
We will use this manifest to deploy to our AKS.

Using your favorite text editor, open the file and edit the line which holds the DNS FQDN of the Azure Container Registry :

``` apiVersion: apps/v1 
kind: Deployment
metadata:
  name: web-deployment
spec:
  selector:
    matchLabels:
      app: staticweb
  replicas: 3
  template:
    metadata:
      labels:
        app: staticweb
    spec:
      containers:
      - name: staticweb
        image: youracrname.azurecr.io/sample/webpage
        imagePullPolicy: Always
        ports:
        - containerPort: 80
```

Change "youracrname" to the name of your newly created Azure Container Registry and save it.

Let's deploy the static webpage into AKS.
Use __kubectl apply -f deployment.yml__ command to fire our app into Kubernetes :

![result](/images/14.png)

Now our application is deployed into AKS.
Let's check where our application landed.

Use __kubectl get pods -n default__ to check our static web pods :

![result](/images/15.png)

Let's deep dive into one of them. I've chosen `web-deployment-6b66dc5d85-52d6w`.
Use `kubectl describe pod [podname]` and check out the __Events__ section :

![result](/images/16.png)

As you can see, our application landed on the __User__ nodepool, which is meant for our application.
Now our cluster configuration is complete.

## OPTIONAL : View the application we've just deployed

If you're curious and you want to see the application,
use this command to make AKS expose your application pods so you can see the web page running :

` kubectl expose deployment web-deployment --type=LoadBalancer --name=web-svc`

This will take sometime to complete.
You can check the progress at `kubectl get svc` :

![result](/images/17.png)

Notice that our web-svc now has an external IP.
Open your favorite web browser and let's browse to see what we have :

![result](/images/18.png)

Congratulations! you've finished this tutorial.
What you've learned :

* Types of nodepools on AKS
* How to use Ephemeral OS on AKS and VM sizes available for it
* How to taint your nodepool only for system workloads
* How to build a Docker Image directly on Azure Container Registry
* How to seamlessly connect between AKS and Azure Container Registry

Please let me know what you think at : nadebu@outlook.com






