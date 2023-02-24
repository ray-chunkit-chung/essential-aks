# essential-aks

This tutorial is based on the following materials:

- The azure-vote app is from <https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/aks/tutorial-kubernetes-prepare-app.md>
- The tutorial flow is from <https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-portal?tabs=azure-cli>
- The aks doc is here <https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/aks/tutorial-kubernetes-scale.md>

## Step 1 Try the azure vote app in localhost

```bash
docker compose -f "docker-compose.yaml" up -d --build 
```

Visit <http://localhost:8080/>

## Login az cli

Create a .env file

```bash
export SUBSCRIPTION="xxxxx"
export TENANT="xxxx"
export LOCATION="xxxx"
export RESOURCE_GROUP="xxxx"
acrName
```

Install az cli

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

```bash
source .env
az login --tenant $TENANT
az account set -s $SUBSCRIPTION
```

## Step 2 create container registry

```bash
source .env
az group create --name $RESOURCE_GROUP --location $LOCATION
az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku $ACR_SKU
az acr login --name $ACR_NAME
export ARC_SERVER="$(az acr list --resource-group $RESOURCE_GROUP --query "[].{acrLoginServer:loginServer}" | jq '.[0].acrLoginServer' | tr -d '"')"
```

## Step 3 Push image to acr

Assume env variable $ARC_SERVER contains the arc server url

```bash
docker tag mcr.microsoft.com/azuredocs/azure-vote-front:v1 $ARC_SERVER/azure-vote-front:v1
docker push $ARC_SERVER/azure-vote-front:v1
```

Check if the repo exists

```bash
az acr repository list --name $ACR_NAME --output table
az acr repository show-tags --name $ACR_NAME --repository azure-vote-front --output table
```

## Step 4 Create aks

Install kubectl

```bash
az aks install-cli
```

Create aks cluster

```bash
source .env
az aks create \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_NAME \
    --node-count 2 \
    --generate-ssh-keys \
    --attach-acr $ACR_NAME

# Configure kubectl to connect to aks
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_NAME
```


```bash
kubectl get nodes
```

## Step 5 Publish app in aks

Assume env variable $ARC_SERVER contains the arc server url. This is used in azure-vote-all-in-one-redis.yaml

Publish app

```bash
kubectl apply -f k8s/azure-vote-all-in-one-redis.yaml
```

Check health status

```bash
kubectl get pods
kubectl get service azure-vote-front --watch
```

## (Optional) Scale pods/nodes manually

Scale pods

```bash
kubectl scale --replicas=5 deployment/azure-vote-front
kubectl get pods
```

Scale nodes

```bash
az aks scale --resource-group $RESOURCE_GROUP --name $AKS_NAME --node-count 3
```

## Step 6 Scale pods automatically

```bash
kubectl apply -f k8s/azure-vote-hpa.yaml
kubectl get hpa
```





## Health monitoring

<https://azure.microsoft.com/en-us/pricing/details/monitor/>

![image](https://user-images.githubusercontent.com/26511618/221086773-e9cc0125-9536-461a-a013-0c5be5ee9d99.png)

https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-overview

![image](https://user-images.githubusercontent.com/26511618/221089022-e96bba6b-9127-4a78-aaa3-2b8993ada83c.png)


## Old notes

Before you add your cluster to Azure Arc, you’ll need:

**To identify or create a Kubernetes cluster**

A new or existing Kubernetes cluster
The cluster must use Kubernetes version 1.13 or later (including OpenShift 4.2 or later and other Kubernetes derivatives). Learn how to create a Kubernetes cluster
Access to ports 443 and 9418

Make sure the cluster has access to these ports, and the required outbound URLs
Connectivity method
You can connect to the internet over a public endpoint, through a proxy server, or over a private endpoint. An Azure Arc Private link scope is required to connect over a private endpoint.

Create an Azure Arc Private link scope

To set up your local machine

Azure CLI
An Azure CLI script will be generated that you will run locally to connect your cluster.
Learn how to install the Azure CLI on your local machine

CLI extensions
Install the latest connectedk8s and k8s-configuration, k8s-extension, customlocation CLI extensions. Learn how to install these extensions

Kubeconfig file with cluster admin permissions
The file should be accessible via your CLI tooling. Learn how to get a kubeconfig file

## Step X Learn how to create a Kubernetes cluster

[Learn how to create a Kubernetes cluster](https://kind.sigs.k8s.io/docs/user/quick-start/#creating-a-cluster)

## Step X Access to ports 443 and 9418

<https://aka.ms/ArcK8sOutboundURLs>

## Step X Create an Azure Arc Private Link Scope

## Step X Local machine install az cli and extensions

connectedk8s and k8s-configuration, k8s-extension, customlocation CLI extensions

## Step X Local machine install cli, extensions

get a kubeconfig file <https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/>

## Finally

Delete everything

```bash
az group delete --yes --name $RESOURCE_GROUP
```
