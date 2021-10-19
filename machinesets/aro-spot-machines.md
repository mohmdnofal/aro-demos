# Create a Machine Set using Spot Virtual Machines in Azure Red Hat OpenShift 
Azure Spot Virtual Machines is a great way to get your VMs at a great discount comparing on the on-demand price by bidding on the un-unsed capacity in a given Azure region. The only caveat is that your VM will be deallocated or deleted if capacity is needed or the current spot prices is higher than your maximum price.

Spot VMs are a great fit for workloads that can tolerate evictions such as dev/test, batch, or any transient workload and if you're eager enough you can run non-transient/long running workloads by having a strategy to use different VM SKUs. 

In this demo we will create a new machine set in Azure Red Hat OpenShift using a Spot Virtual Machine. 

For more information regarding Azure Spot Virtual Machines please visit https://docs.microsoft.com/en-us/azure/virtual-machines/spot-vms


# Create Spot Machine Set Demo
OpenShift allows you to create Machine Sets using spot VMs, more information can be found [here](https://docs.openshift.com/container-platform/4.7/machine_management/creating_machinesets/creating-machineset-azure.html#machineset-creating-non-guaranteed-instance_creating-machineset-azure). 
The whole trick is in using "spotVMOptions" in the "ProviderSpec" section in your  Machine Set definition.
```yaml 
providerSpec:
  value:
    spotVMOptions: {}
```
The above option will default the maximum bid price to "-1", which means that you're willing to pay up to the standard VM on-demand price, this option helps with avoiding evictions for pricing reason. you can always set "maxPrice" if you want a cap on what you're willing to pay. 


#### Note 
This is maybe not so relevant when using Spot VMs but its worth noting that Machine Sets in OpenShift are scoped to a single Availability Zone.The below example creates a single machine set only in 1 AZ, in production you will need to create 3 machine sets across 3 AZs to ensure high availability.

```shell 
#Lets assume that you have a new workload which requires memory, and we need to dedicate some new nodes to it, so we decided that "Standard_E4s_v3" VM SKU is a good fit and we will use Spot VMs. 

#Create a new Machine Set definition from your template and modfiy the right parameters, I created one for convenience "machineset-spot-zone1.yaml" 

#create the new machineset 
kubectl apply -f machineset-spot-zone1.yaml

#validate the machineset being created (it will take ~5 minutes for a new machine to be added)
kubectl get machinesets.machine.openshift.io -n openshift-machine-api
NAME                                              DESIRED   CURRENT   READY   AVAILABLE   AGE
arodemo-northeurope-c-b24pq-worker-northeurope1   1         1         1       1           6d3h
arodemo-northeurope-c-b24pq-worker-northeurope2   1         1         1       1           6d3h
arodemo-northeurope-c-b24pq-worker-northeurope3   1         1         1       1           6d3h
memory-node-zone1                                 1         1         1       1           47m
spot-node-zone1                                   1         1                             10s


#have a closer look at the progress 
kubectl describe machinesets spot-node-zone1 -n openshift-machine-api

#Check the events, dont worry if you see a "Warning" which indicates "FailedCreate" thats just the re-conciler detecting that there is no machine and triggering the creation of a new one 
kubectl get events -n openshift-machine-api

#Check and watch until the vms gets created 
kubectl get nodes -w
NAME                                                    STATUS   ROLES    AGE    VERSION
arodemo-northeurope-c-b24pq-master-0                    Ready    master   6d3h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-master-1                    Ready    master   6d3h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-master-2                    Ready    master   6d3h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-worker-northeurope1-hrs4g   Ready    worker   6d3h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-worker-northeurope2-jl7hj   Ready    worker   6d3h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-worker-northeurope3-zt4d9   Ready    worker   6d3h   v1.21.1+9807387
memory-node-zone1-q6df4                                 Ready    worker   48m    v1.21.1+9807387
spot-node-zone1-kwbcn                                   Ready    worker   84s    v1.21.1+9807387


#lets validate if the VM was created using the spot pricing we defined (remember we are using the default "-1"), we need to list the VMs in the ARO "nodes resource group" so we will borrow the same parameters we defined before during the cluster creation 
LOCATION=northeurope        
CLUSTER_NAME="arodemo-$LOCATION-cluster"
RG="$CLUSTER_NAME-master-rg"
NODES_RG="$CLUSTER_NAME-nodes-rg"


az vm list \
   -g $NODES_RG \
   --query '[].{Name:name, MaxPrice:billingProfile.maxPrice}' \
   --output table

Name                                                   MaxPrice
-----------------------------------------------------  ----------
arodemo-northeurope-c-b24pq-master-0
arodemo-northeurope-c-b24pq-master-1
arodemo-northeurope-c-b24pq-master-2
arodemo-northeurope-c-b24pq-worker-northeurope1-hrs4g
arodemo-northeurope-c-b24pq-worker-northeurope2-jl7hj
arodemo-northeurope-c-b24pq-worker-northeurope3-zt4d9
memory-node-zone1-q6df4
spot-node-zone1-kwbcn                                  -1.0


#Clean Up
kubectl scale --replicas=0 machineset spot-node-zone1 -n openshift-machine-api


#this concludes the demo 
``` 
