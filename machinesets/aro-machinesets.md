# Working with Machine Sets in Azure Red Hat Open Shift (ARO)

Machine Sets in Open Shift is what node pools are to Kubernetes, its a group of machines sharing the same VM SKU. in Open Shift machine sets are defined programmatically using the Machine API, the most important thing to know is that machine sets are always scoped to a single availability zone, if you need multiple availability zones support for you would need to create multiple machine sets each dedicated to a single AZ. 
For more information please read the [machine sets docs](https://docs.openshift.com/container-platform/4.7/machine_management/creating_machinesets/creating-machineset-azure.html#machineset-creating-non-guaranteed-instance_creating-machineset-azure).

A default installation for ARO will create 3 machine sets, 1 machineset per availability zone assuming you installed ARO in a region which supports availability zones, if you installed ARO in a region with no AZ support you will get 1 machine set with all the machines inside it.

In this demo the scenario is we received a request for a new workload which requires mainly memory, so we decided to dedicate a specific machine set for this workload, we looked into the [supported VMs in ARO](https://docs.microsoft.com/en-gb/azure/openshift/support-policies-v4#supported-virtual-machine-sizes) and we decided that "Standard_E4s_v3" is a suitable SKU. We also decided that we will add a label to all nodes created by this machine set so we can use affinity and autoscaling in the future, the label is "machineset-name=memory-node-zone1", you can add as many labels as needed to a given machine set definition which is a recommended approach. lastly the machine will be created in Zone 1. 

#### Note 
Machine Sets in OpenShift are scoped to a single Availability Zone.The below example creates a single machine set only in 1 AZ, in production you will need to create 3 machine sets across 3 AZs to ensure high availability.



# Create a machine set in ARO using on-demand pricing Demo 

```shell 
#view existing machine sets 
kubectl get machinesets.machine.openshift.io -n openshift-machine-api

NAME                                              DESIRED   CURRENT   READY   AVAILABLE   AGE
arodemo-northeurope-c-b24pq-worker-northeurope1   1         1         1       1           6d2h
arodemo-northeurope-c-b24pq-worker-northeurope2   1         1         1       1           6d2h
arodemo-northeurope-c-b24pq-worker-northeurope3   1         1         1       1           6d2h

#validate the spread of the nodes across availability zones 
kubectl describe nodes -l node-role.kubernetes.io/worker= | grep zone 
                    failure-domain.beta.kubernetes.io/zone=northeurope-1
                    topology.kubernetes.io/zone=northeurope-1
                    failure-domain.beta.kubernetes.io/zone=northeurope-2
                    topology.kubernetes.io/zone=northeurope-2
                    failure-domain.beta.kubernetes.io/zone=northeurope-3
                    topology.kubernetes.io/zone=northeurope-3

#View the configuration for an existing machineset 
kubectl get machinesets.machine.openshift.io arodemo-northeurope-c-b24pq-worker-northeurope1 -n openshift-machine-api -o yaml

#now export the configuration of the machineset so we can use it as a template to create new machine sets 
kubectl get machinesets.machine.openshift.io arodemo-northeurope-c-b24pq-worker-northeurope1 -n openshift-machine-api -o yaml > machineset_template.yaml

#remove all the runtime metadata from the machine set template file, I already updated mine "machineset_template.yaml" 

#Create a new file from your template and give it a descriptive name, I've created one with annotations for convenience "machineset-memory-zone1.yaml" 

#create the new machineset 
kubectl apply -f machineset-memory-zone1.yaml

#validate the machineset being created (it will take ~5 minutes for a new machine to be added)
kubectl get machinesets.machine.openshift.io -n openshift-machine-api
NAME                                              DESIRED   CURRENT   READY   AVAILABLE   AGE
arodemo-northeurope-c-b24pq-worker-northeurope1   1         1         1       1           6d2h
arodemo-northeurope-c-b24pq-worker-northeurope2   1         1         1       1           6d2h
arodemo-northeurope-c-b24pq-worker-northeurope3   1         1         1       1           6d2h
memory-node-zone1                                 1         1                             73s


#have a closer look at the progress 
kubectl describe machinesets memory-node-zone1 -n openshift-machine-api

#Check the events, dont worry if you see a "Warning" which indicates "FailedCreate" that's just the reconciler detecting that there is no machine and triggering the creation of a new one 

kubectl get events -n openshift-machine-api

2m18s       Warning   FailedCreate       machine/memory-node-zone1-c2t4c                                     CreateError: failed to reconcile machine "memory-node-zone1-c2t4c"s: failed to create vm memory-node-zone1-c2t4c: failed to create or get machine: compute.VirtualMachinesCreateOrUpdateFuture: asynchronous operation has not completed
26s         Normal    Updated            machine/memory-node-zone1-c2t4c                                     Updated machine "memory-node-zone1-c2t4c"

#Check again and watch until the machine gets created 
kubectl get nodes -w
NAME                                                    STATUS   ROLES    AGE    VERSION
arodemo-northeurope-c-b24pq-master-0                    Ready    master   6d3h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-master-1                    Ready    master   6d3h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-master-2                    Ready    master   6d3h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-worker-northeurope1-hrs4g   Ready    worker   6d2h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-worker-northeurope2-jl7hj   Ready    worker   6d2h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-worker-northeurope3-zt4d9   Ready    worker   6d2h   v1.21.1+9807387
memory-node-zone1-c2t4c                                 Ready    worker   2m18s   v1.21.1+9807387

#check the labels on the node that we created, verify that the label "machineset-name=memory-node-zone1" was added 
kubectl get nodes memory-node-zone1-c2t4c --show-labels

NAME                      STATUS   ROLES    AGE   VERSION           LABELS
memory-node-zone1-c2t4c   Ready    worker   31s   v1.21.1+9807387   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=Standard_E4s_v3,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=northeurope,failure-domain.beta.kubernetes.io/zone=northeurope-1,kubernetes.io/arch=amd64,kubernetes.io/hostname=memory-node-zone1-c2t4c,kubernetes.io/os=linux,machineset-name=memory-node-zone1,node-role.kubernetes.io/worker=,node.kubernetes.io/instance-type=Standard_E4s_v3,node.openshift.io/os_id=rhcos,topology.kubernetes.io/region=northeurope,topology.kubernetes.io/zone=northeurope-1

#Manually scale a machine set 
kubectl scale --replicas=2 machineset memory-node-zone1 -n openshift-machine-api

#Validate the machine set 
kubectl get machineset memory-node-zone1 -n openshift-machine-api -w
NAME                DESIRED   CURRENT   READY   AVAILABLE   AGE
memory-node-zone1   2         2         1       1           12m

memory-node-zone1   2         2         2       2           16m


#check the nodes 
kubectl get nodes 
NAME                                                    STATUS   ROLES    AGE    VERSION
arodemo-northeurope-c-b24pq-master-0                    Ready    master   6d3h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-master-1                    Ready    master   6d3h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-master-2                    Ready    master   6d3h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-worker-northeurope1-hrs4g   Ready    worker   6d2h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-worker-northeurope2-jl7hj   Ready    worker   6d2h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-worker-northeurope3-zt4d9   Ready    worker   6d2h   v1.21.1+9807387
memory-node-zone1-g625z                                 Ready    worker   76s    v1.21.1+9807387
memory-node-zone1-c2t4c                                 Ready    worker   13m    v1.21.1+9807387

#If you want to demo autoscaling, then just scale down this machine set to 1 node again, otherwise skip to the next step which is scale down to zero 
kubectl scale --replicas=1 machineset memory-node-zone1 -n openshift-machine-api


#you can also scale a machine set to zero and scale it again when needed 
kubectl scale --replicas=0 machineset memory-node-zone1 -n openshift-machine-api


#this concludes this demo
``` 
