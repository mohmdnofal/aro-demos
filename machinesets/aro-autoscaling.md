# Autoscaling in Azure Red Hat OpenShift 

Nodes Autoscaling in Kubernetes Distributions including OpenShift is done using the [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) upstream project, which increases the size of a given cluster based on "Pods Pending Scheduling" in other words when you don't have sufficient resources in your cluster to schedule more pods, the scale in happens when the CA detects that some nodes are consistently not needed by being underutilized. 

In OpenShift to enable autoscaling you need to follow 2 steps:
1- Create the Cluster Autoscaler object which is scoped to the cluster 
2- Enable autoscaling on individual machine sets which require autoscaling using the machine autoscaler API 

More Information can be found here https://docs.openshift.com/container-platform/4.8/machine_management/applying-autoscaling.html

# Enable Autoscaling in ARO demo 

We will follow the same steps outlined above, create the autoscaler then enable scaling on an existing machine set. I created a cluster autoscaler for this demo specifically which maintains 3 worker nodes as minimum and scales out to 6 worker nodes as a max. Then I enabled autoscaling using the machine autoscaler API on the "memory-node-zone1" machine set we created earlier. Finally to see autoscaling in action I created a deployment which targets machines created by this machine set using "nodeSelector", the deployment will ask for more resources than that in the node as such autoscaling will be triggered. 

```shell 
#create the cluster autoscaler object (I created one for convinience) this one is scoped to this demo 
kubectl apply -f aro-clusterautoscaler.yaml

#validate the configuration 
kubectl describe clusterautoscalers.autoscaling.openshift.io default 

Name:         default
Namespace:    
Labels:       <none>
Annotations:  <none>
API Version:  autoscaling.openshift.io/v1
Kind:         ClusterAutoscaler
...
Spec:
  Pod Priority Threshold:  -10
  Resource Limits:
    Cores:
      Max:  100
      Min:  24
    Gpus:
      Max:            16
      Min:            0
      Type:           nvidia.com/gpu
      Max:            4
      Min:            0
      Type:           amd.com/gpu
    Max Nodes Total:  10
    Memory:
      Max:  250
      Min:  48
  Scale Down:
    Delay After Add:      10m
    Delay After Delete:   1m
    Delay After Failure:  30s
    Enabled:              true
    Unneeded Time:        5m
Events:                   <none>


#Now that the cluster autoscaler has been enabled on the cluster level, we need to enable it on a machine set (note you can apply it to all machine sets) but we are going to do one only for sake of this demo 


#lets get our existing machinesets 
kubectl get machinesets.machine.openshift.io -n openshift-machine-api

NAME                                              DESIRED   CURRENT   READY   AVAILABLE   AGE
arodemo-northeurope-c-b24pq-worker-northeurope1   1         1         1       1           7d8h
arodemo-northeurope-c-b24pq-worker-northeurope2   1         1         1       1           7d8h
arodemo-northeurope-c-b24pq-worker-northeurope3   1         1         1       1           7d8h
memory-node-zone1                                 1         1         1       1           29h
spot-node-zone1                                   0         0                             28h


#I created "aro-memory-zone1-machine-scaler.yaml" for this demo, this configuration targets the "memory-node-zone1" machine set and scales between min 1 and max 3 nodes 
kubectl apply -f aro-memory-zone1-machine-scaler.yaml

#validate what you created, you can see that the min number of nodes is set to 1 and the max is set to 3 
kubectl get machineautoscalers -n openshift-machine-api

NAME                REF KIND     REF NAME            MIN   MAX   AGE
memory-node-zone1   MachineSet   memory-node-zone1   1     3     12s

#lets test our configuration by trying to overload the machineset
#We will create a deployment and assign it to the machine set we enabled autoscaling on, as we enabled the autoscaling on the "memory-node-zone1" we can use the "machineset-name: memory-node-zone1" key as our affinity key in our deployment, you can view this by showing the lable of your machine
kubectl get nodes memory-node-zone1-c2t4c --show-labels 

NAME                      STATUS   ROLES    AGE   VERSION           LABELS
memory-node-zone1-c2t4c   Ready    worker   31m   v1.21.1+9807387   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=Standard_E4s_v3,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=northeurope,failure-domain.beta.kubernetes.io/zone=northeurope-1,kubernetes.io/arch=amd64,kubernetes.io/hostname=memory-node-zone1-c2t4c,kubernetes.io/os=linux,machineset-name=memory-node-zone1,node-role.kubernetes.io/worker=,node.kubernetes.io/instance-type=Standard_E4s_v3,node.openshift.io/os_id=rhcos,topology.kubernetes.io/region=northeurope,topology.kubernetes.io/zone=northeurope-1


#I have created a simple Nginx Deployment which requires 8 replicas, each replica requires 1vCPU, this should trigger the autoscaler to start adding machines. the deployment has a nodeSelector "machineset-name: memory-node-zone1" so we ensure we are targeting the machine set of concern
kubectl apply -f nginx_deployment.yaml

#get the pods, you should see some being created and others are pending (keep this one open in a terminal)
kubectl get pods -o wide -w 
NAME                               READY   STATUS              RESTARTS   AGE   IP       NODE                      NOMINATED NODE   READINESS GATES
nginx-deployment-664d4db75-7sgbh   0/1     ContainerCreating   0          10s   <none>   memory-node-zone1-c2t4c   <none>           <none>
nginx-deployment-664d4db75-9dj4l   0/1     Pending             0          10s   <none>   <none>                    <none>           <none>
nginx-deployment-664d4db75-bwdl5   0/1     ContainerCreating   0          10s   <none>   memory-node-zone1-c2t4c   <none>           <none>
nginx-deployment-664d4db75-j5wkz   0/1     Pending             0          10s   <none>   <none>                    <none>           <none>
nginx-deployment-664d4db75-l2xbv   0/1     Pending             0          10s   <none>   <none>                    <none>           <none>
nginx-deployment-664d4db75-rjfnr   0/1     Pending             0          10s   <none>   <none>                    <none>           <none>
nginx-deployment-664d4db75-wn92k   0/1     Pending             0          10s   <none>   <none>                    <none>           <none>
nginx-deployment-664d4db75-z7vfj   0/1     ContainerCreating   0          10s   <none>   memory-node-zone1-c2t4c   <none>           <none>
nginx-deployment-664d4db75-bwdl5   1/1     Running             0          13s   10.0.24.7   memory-node-zone1-c2t4c   <none>           <none>
nginx-deployment-664d4db75-z7vfj   1/1     Running             0          13s   10.0.24.8   memory-node-zone1-c2t4c   <none>           <none>
nginx-deployment-664d4db75-7sgbh   1/1     Running             0          13s   10.0.24.9   memory-node-zone1-c2t4c   <none>           <none>
#open another terminal window check the cluster autoscaler logs, you should see scaleUp actions being triggered to add nodes 
kubectl get configmaps cluster-autoscaler-status -n openshift-machine-api -o yaml -w

apiVersion: v1
data:
  status: |+
    Cluster-autoscaler status at 2021-10-06 20:30:12.218337715 +0000 UTC:
    Cluster-wide:
      Health:      Healthy (ready=7 unready=0 notStarted=0 longNotStarted=0 registered=7 longUnregistered=0)
                   LastProbeTime:      2021-10-06 20:30:11.009025266 +0000 UTC m=+571.771061426
                   LastTransitionTime: 2021-10-06 20:21:10.526711772 +0000 UTC m=+31.288748032
      ScaleUp:     InProgress (ready=7 registered=7)
                   LastProbeTime:      2021-10-06 20:30:11.009025266 +0000 UTC m=+571.771061426
                   LastTransitionTime: 2021-10-06 20:29:46.561893828 +0000 UTC m=+547.323930088
      ScaleDown:   NoCandidates (candidates=0)
                   LastProbeTime:      2021-10-06 20:30:11.009025266 +0000 UTC m=+571.771061426
                   LastTransitionTime: 2021-10-06 20:21:10.526711772 +0000 UTC m=+31.288748032

    NodeGroups:
      Name:        MachineSet/openshift-machine-api/memory-node-zone1
      Health:      Healthy (ready=1 unready=0 notStarted=0 longNotStarted=0 registered=1 longUnregistered=0 cloudProviderTarget=3 (minSize=1, maxSize=3))
                   LastProbeTime:      2021-10-06 20:30:11.009025266 +0000 UTC m=+571.771061426
                   LastTransitionTime: 2021-10-06 20:25:50.738014659 +0000 UTC m=+311.500050919
      ScaleUp:     InProgress (ready=1 cloudProviderTarget=3)
                   LastProbeTime:      2021-10-06 20:30:11.009025266 +0000 UTC m=+571.771061426
                   LastTransitionTime: 2021-10-06 20:29:46.561893828 +0000 UTC m=+547.323930088
      ScaleDown:   NoCandidates (candidates=0)
                   LastProbeTime:      2021-10-06 20:30:11.009025266 +0000 UTC m=+571.771061426
                   LastTransitionTime: 2021-10-06 20:25:50.738014659 +0000 UTC m=+311.500050919

kind: ConfigMap
metadata:
  annotations:
    cluster-autoscaler.kubernetes.io/last-updated: 2021-10-06 20:30:12.218337715 +0000
      UTC
  creationTimestamp: "2021-10-06T20:21:00Z"
  name: cluster-autoscaler-status
  namespace: openshift-machine-api
  resourceVersion: "3842193"
  uid: f60424a6-ba13-49fe-934d-886a2c7be256
---
....

# Check your machine set, you should notice that the desired number has changed to 3 
kubectl get machinesets.machine.openshift.io -n openshift-machine-api

NAME                                              DESIRED   CURRENT   READY   AVAILABLE   AGE
arodemo-northeurope-c-b24pq-worker-northeurope1   1         1         1       1           7d8h
arodemo-northeurope-c-b24pq-worker-northeurope2   1         1         1       1           7d8h
arodemo-northeurope-c-b24pq-worker-northeurope3   1         1         1       1           7d8h
memory-node-zone1                                 3         3         1       1           29h
spot-node-zone1                                   0         0                             29h

#If you check the nodes you should see nodes being added after couple of minutes, wait until the status change from Not Ready to Ready 
kubectl get nodes -w

NAME                                                    STATUS   ROLES    AGE    VERSION
arodemo-northeurope-c-b24pq-master-0                    Ready    master   7d8h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-master-1                    Ready    master   7d8h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-master-2                    Ready    master   7d8h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-worker-northeurope1-hrs4g   Ready    worker   7d8h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-worker-northeurope2-jl7hj   Ready    worker   7d8h   v1.21.1+9807387
arodemo-northeurope-c-b24pq-worker-northeurope3-zt4d9   Ready    worker   7d8h   v1.21.1+9807387
memory-node-zone1-c2t4c                                 Ready    worker   39m    v1.21.1+9807387
memory-node-zone1-r5ksc                                 Ready    worker   36s    v1.21.1+9807387
memory-node-zone1-tpxbm                                 Ready    worker   100s   v1.21.1+9807387

#Once the nodes are all ready, get back to your pods and you should see them all running 
kubectl get pods                                                                    

NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-664d4db75-7sgbh   1/1     Running   0          6m26s
nginx-deployment-664d4db75-9dj4l   1/1     Running   0          6m26s
nginx-deployment-664d4db75-bwdl5   1/1     Running   0          6m26s
nginx-deployment-664d4db75-j5wkz   1/1     Running   0          6m26s
nginx-deployment-664d4db75-l2xbv   1/1     Running   0          6m26s
nginx-deployment-664d4db75-rjfnr   1/1     Running   0          6m26s
nginx-deployment-664d4db75-wn92k   1/1     Running   0          6m26s
nginx-deployment-664d4db75-z7vfj   1/1     Running   0          6m26s

#Now that we have demonestrated how scale out works we can delete the deployment, you can follow the same steps above to check how scale in is taking place 
kubectl delete -f nginx_deployment.yaml


#This concludes the demo 

```