In this demo we will install an Azure Red Hat OpenShift cluster using a custom domain.



1. First you need to register the provider to your subscription 
az provider register -n Microsoft.RedHatOpenShift --wait
az provider register -n Microsoft.Compute --wait
az provider register -n Microsoft.Storage --wait

2. Get the pull secret so you can use the Red Hat Cluster Manager Portal, [instructions here](https://docs.microsoft.com/en-gb/azure/openshift/tutorial-create-cluster#get-a-red-hat-pull-secret-optional)

3. Now we need to define the parameters to make the installation easier and repeatable 

```shell
LOCATION=northeurope        #the location of your cluster
CLUSTER_NAME="aro-customdomain-$LOCATION-cluster"        #the name of your cluster
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
DOMAIN="example.com"
```


4. Now we need to prepare the environment for ARO 

```shell
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



#Create service principal to be used by the cluster
SP_NAME=arone`date +"%d%m%y"`
SP_CLIENT_SECRET=`az ad sp create-for-rbac --name $SP_NAME --skip-assignment --query 'password' -o tsv`
SP_CLIENT_ID=`az ad sp list --display-name $SP_NAME --query '[0].appId' -o tsv`

#get the resource group ID 
RG_ID=$(az group show -n $RG  --query id -o tsv)

### get the vnet ID
VNETID=$(az network vnet show -g $RG --name $ARO_VNET_NAME --query id -o tsv)

# Assign SP Permission to RG 
az role assignment create --assignee $SP_CLIENT_ID --scope $RG_ID --role Contributor
# Assign SP Permission to VNET
az role assignment create --assignee $SP_CLIENT_ID --scope $VNETID --role Contributor

# View Role Assignment
az role assignment list --assignee $SP_CLIENT_ID --all -o table
```


5. Time to create the cluster, this should take ~40 minutes, we need to wait until its done so we can do the rest of the setup 

```shell
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
  --pull-secret @pull-secret.txt \
  --domain $DOMAIN
```


7. As you're creating a cluster with a custom domain, you will need to create the API and Ingress A records in your own domain, assuming you're using Azure DNS, 
then you can make use of the below steps 

```shell 
#get the API server IP 
ARO_API_SERVER_IP=$(az aro show -n $CLUSTER_NAME -g $RG --query '{api:apiserverProfile.ip}' -o tsv)
ARO_INGRESS_IP=$(az aro show -n $CLUSTER_NAME -g $RG --query '{ingress:ingressProfiles[0].ip}' -o tsv)

#create 2 DNS records for your cluster one for the API Server (api) and another for the ingress (*.apps)
DNS_RG=dns_rg
DOMAIN=$DOMAIN
az network dns record-set a add-record -g $DNS_RG -z $DOMAIN -n api -a $ARO_API_SERVER_IP
az network dns record-set a add-record -g $DNS_RG -z $DOMAIN -n "*.apps" -a $ARO_INGRESS_IP

#you can try to access the admin console now, you don't have to login, but notice that you will get a warnning that the page isn't secure as the browser can't verify the self signed certificate provided from OpenShift by default
#below command will return the console URL, open it in your browser 
az aro show \
    --name $CLUSTER_NAME \
    --resource-group $RG \
    --query "consoleProfile.url" -o tsv
```

8. Now we have the domain configured, we need to use our own certificates, and upload them to the OpenShift Ingress and API Server. the part of creating the certificate is only needed assuming you don't have one already. 

```shell
#I'm using certbot to create a certificate signed by LetEncrypt, you can skip this if you have your certificate already along with the keys 
Find the installation method suitable for your operating system, Im using a Mac, instructurions here https://certbot.eff.org/


#create the certificate, you will be prompted to create a record in your DNS server, the installation will wait until you create it, don't proceed with the installation until the record was created and is resolvable 
sudo ./certbot certonly \
--manual \
-d "*.$DOMAIN" \
-d "$DOMAIN" \
-d "*.apps.$DOMAIN" \
--preferred-challenges dns \
--server https://acme-v02.api.letsencrypt.org/directory \
--agree-tos \
--email youremail@mail.com


#open another shell and add the requested records, instructions below for Azure DNS 
DNS_RG=YOUR_DNS_RG
DOMAIN=$DOMAIN
az network dns record-set txt add-record -g $DNS_RG -z $DOMAIN -n "_acme-challenge" -v "COPY_FROM_THE_CERTIFICATE_COMMAND_ABOVE" --ttl 60

#verify that you can resolve the record, once it returns the correct value, proceed with the installation. 
dig _acme-challenge.$DOMAIN ANY


#The installation will proceed
If you are using a Mac then you will find your records under /etc/letsencrypt/live/$DOMAIN/ 


#Get the Kube Admin Password
KUBE_ADMIN_PASSWORD=$(az aro list-credentials -g "$RG" -n "$CLUSTER_NAME" --query kubeadminPassword -o tsv)

#the admin username is default 
KUBE_ADMIN_USERNAME=kubeadmin

#get the API Server FQDN 
apiServer=$(az aro show -n $CLUSTER_NAME -g $RG --query apiserverProfile.url -o tsv)

#now we need to login so we can create the certificate and path the ingress and the API server 
oc login $apiServer -u $KUBE_ADMIN_USERNAME -p $KUBE_ADMIN_PASSWORD


#we start first by creating a secret for the ingress and then we patch it with the new certificate 
oc create secret tls ingress-certs --cert=${CERTDIR}/fullchain.pem --key=${CERTDIR}/privkey.pem -n openshift-ingress
oc patch ingresscontroller default -n openshift-ingress-operator --type=merge --patch='{"spec": { "defaultCertificate": { "name": "ingress-certs" }}}'

#In order for the patch to get applied the ingress pods will be redeployed, watch for them and make sure they are running 
kubectl get pods -n openshift-ingress


#now go to your console and check the certificate there, you shold see that its secure and signed. 
az aro show \
    --name $CLUSTER_NAME \
    --resource-group $RG \
    --query "consoleProfile.url" -o tsv


#you can copy, and paste your password in the console 
echo $KUBE_ADMIN_PASSWORD | pbcopy



#Lets repeat the same for the API server 
oc create secret tls api-certs --cert=${CERTDIR}/fullchain.pem --key=${CERTDIR}/privkey.pem -n openshift-config
oc patch apiserver cluster --type=merge --patch='{"spec": {"servingCerts": {"namedCertificates": [{"names": [" '$LE_API' "], "servingCertificate": {"name": "api-certs"}}]}}}'

#this would take a while, check below and wait until at least you have 2 replicas running 
kubectl get pods -n openshift-kube-apiserver 

#login again to your API server 
oc login $apiServer -u $KUBE_ADMIN_USERNAME -p $KUBE_ADMIN_PASSWORD
oc get nodes 
```


9. "Optional" integrate with Azure Monitor 

```shell

# you can follow the docs here, or the below instrcutions 
https://docs.microsoft.com/en-gb/azure/azure-monitor/insights/container-insights-azure-redhat4-setup

#Azure monitor team has provided a script to install and configure the monitoring for your ARO 4 cluster 
curl -o enable-monitoring.sh -L https://aka.ms/enable-monitoring-bash-script

#get the kube config context
kubeContext=$(oc config current-context)


#create a log analytics workspace or use an existing one 
to create a log analytics workspace https://docs.microsoft.com/en-gb/azure/azure-monitor/learn/quick-create-workspace

#get your workspace resource ID from here
az resource list --resource-type Microsoft.OperationalInsights/workspaces -o json


#fill in the below values 
export azureAroV4ClusterResourceId=$(az aro show -g $RG -n $CLUSTER_NAME -o tsv --query id)
export logAnalyticsWorkspaceResourceId="/subscriptions/<subscriptionId>/resourceGroups/<resourceGroupName>/providers/microsoft.operationalinsights/workspaces/<workspaceName>"
export kubeContext=$kubeContext  


#enable monitoring (it will take ~15 minutes to start viewing the metrics)
bash enable-monitoring.sh --resource-id $azureAroV4ClusterResourceId --kube-context $kubeContext --workspace-id $logAnalyticsWorkspaceResourceId
``` 

10. When you're done with your testing scenarios and you want to clean up, then below how to delete the cluster 
```shell 
#Delete the cluster
az aro delete -g "$RG" -n "$CLUSTER"
``` 
