# Deploy an Azure Red Hat OpenShift Cluster 
In this demo we will install a standard Azure Red Hat OpenShift cluster with 3 worker nodes, this cluster will be later used for the machine set exercise. If you have an existing demo cluster feel free to skip this part. 



# Deploy an ARO cluster demo 

```shell 
#Define your parameters 
LOCATION=northeurope        #the location of your cluster
CLUSTER_NAME="arodemo-$LOCATION-cluster"        #the name of your cluster
RG="$CLUSTER_NAME-master-rg"    #the name of the resource group where you want to create your cluster
NODES_RG="$CLUSTER_NAME-nodes-rg"
ARO_VNET_NAME="$CLUSTER_NAME-vnet"
ARO_VNET_CIDR="192.168.0.0/16"
ARO_MASTERS_SUBNET_NAME=$CLUSTER_NAME-master-subnet
ARO_MASTERS_SUBNET_CIDR=192.168.0.0/24
ARO_WORKERS_SUBNET_NAME=$CLUSTER_NAME-workers-subnet
ARO_WORKERS_SUBNET_CIDR=192.168.1.0/24
POD_CIDR=10.0.0.0/16
SERVICE_CIDR=172.16.0.0/16
API_SERVER_VISIBILITY=Public
INGRESS_VISIBILITY=Public
WORKER_COUNT=3


#Create or re-use an existing pull secret 
Follw instcructions here https://docs.microsoft.com/en-gb/azure/openshift/tutorial-create-cluster#get-a-red-hat-pull-secret-optional


#create a resource group 
az group create -g "$RG" -l $LOCATION


#create the VNET for ARO 
az network vnet create \
  -g "$RG" \
  -n $ARO_VNET_NAME \
  --address-prefixes $ARO_VNET_CIDR \
  >/dev/null


#create 2 subnets one for the masters and another one for the nodes 

az network vnet subnet create \
  --name $ARO_MASTERS_SUBNET_NAME \
  --resource-group $RG \
  --vnet-name $ARO_VNET_NAME   \
  --address-prefix $ARO_MASTERS_SUBNET_CIDR \
  --service-endpoints Microsoft.ContainerRegistry

az network vnet subnet create \
  --name $ARO_WORKERS_SUBNET_NAME \
  --resource-group $RG \
  --vnet-name $ARO_VNET_NAME   \
  --address-prefix $ARO_WORKERS_SUBNET_CIDR \
  --service-endpoints Microsoft.ContainerRegistry

#disable network policies on the masters subnet 
az network vnet subnet update \
  -g "$RG" \
  --vnet-name $ARO_VNET_NAME \
  -n "$ARO_MASTERS_SUBNET_NAME" \
  --disable-private-link-service-network-policies true \
  >/dev/null



#Create service principal to be used with the cluster 
SP_NAME=arodemo`date +"%d%m%y"`
SP_CLIENT_SECRET=`az ad sp create-for-rbac --name $SP_NAME --skip-assignment --query 'password' -o tsv`
SP_CLIENT_ID=`az ad sp list --display-name $SP_NAME --query '[0].appId' -o tsv`

# Assign the SP Permission to the subcription, this can be scoped to the resource group only, but kept like this for simplicity of the demo 
az role assignment create --assignee $SP_CLIENT_ID  --role "Contributor"
az role assignment create --assignee $SP_CLIENT_ID  --role "User Access Administrator"

# View Role Assignment
az role assignment list --assignee $SP_CLIENT_ID --all -o table




#create the cluster 

az aro create \
  -g "$RG" \
  -n "$CLUSTER_NAME" \
  --vnet $ARO_VNET_NAME \
  --master-subnet "$ARO_MASTERS_SUBNET_NAME" \
  --worker-subnet "$ARO_WORKERS_SUBNET_NAME" \
  --worker-count $WORKER_COUNT \
  --cluster-resource-group $NODES_RG \
  --pod-cidr $POD_CIDR \
  --service-cidr $SERVICE_CIDR \
  --apiserver-visibility $API_SERVER_VISIBILITY\
  --ingress-visibility $INGRESS_VISIBILITY\
  --client-id $SP_CLIENT_ID\
  --client-secret $SP_CLIENT_SECRET\
  --pull-secret @pull-secret.txt 



#list clusters
az aro list -o table



#get the admin creds to login to the cluster 
#Get the Kube Admin Password
KUBE_ADMIN_PASSWORD=$(az aro list-credentials -g "$RG" -n "$CLUSTER_NAME" --query kubeadminPassword -o tsv)

#the admin username is default 
KUBE_ADMIN_USERNAME=kubeadmin

#get the API Server FQDN 
apiServer=$(az aro show -n $CLUSTER_NAME -g $RG --query apiserverProfile.url -o tsv)

#now we need to login so we can create the certificate and path the ingress and the API server 
oc login $apiServer -u $KUBE_ADMIN_USERNAME -p $KUBE_ADMIN_PASSWORD

#validate 
kubectl get nodes

```