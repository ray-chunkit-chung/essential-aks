# essential-aks

## Reference

This tutorial is based on the following materials:

- The azure-vote app is from <https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/aks/tutorial-kubernetes-prepare-app.md>
- The tutorial flow is from <https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-portal?tabs=azure-cli>
- The aks doc is here <https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/aks/tutorial-kubernetes-scale.md>

## Summary of gke vs aks vs eks 2022

https://komodor.com/blog/the-2022-managed-kubernetes-showdown-gke-vs-aks-vs-eks/#:~:text=The%20key%20difference%20relates%20to,cluster%20for%20the%20control%20plane.


## An insight of Istio

<https://qiita.com/hiro8ma/items/1d622deffa3f8e7520c2> Summarizing by google-translating:

(Image from hiro8ma qiita)
![image](https://user-images.githubusercontent.com/26511618/220852382-e10ff90d-f00f-45e0-ac56-1546c7c76fcf.png)


### Microservice challenging points

- Dependency resolution -- Needs to be properly load balanced
- Communication Observability -- Need to consider how much impact a fault that occurs in a service has. Need to observe whether inter-service communication health and to what extent inter-service communication is performed
- Authentication/Authorization -- When implementing multiple services, one may need to limit the authority of one service to call another service's specific APIs. As with collaboration, it is necessary to correctly manage encryption keys and authority
- Fault isolation -- Communication between microservices is more likely to fail than communication by process, so it is necessary to set the retry count, timeout, and upper limit of retry correctly

### Addressing microservice challenging points

- Deploying service proxies (service mesh) as sidecars for each microservice
- No need to write retry processing and circuit breaker code on the application side
- All communication between services goes through the service proxy

### Basic service mesh concepts

<https://istio.io/latest/docs/ops/deployment/architecture/arch.svg>

- In networking, a plane is an abstract conception of where certain processes take place
- A control plane is the part of a network that controls how data packets are forwarded. It manages and configures the proxies to route traffic.
- A data plane (forwarding plane) actually forwards the packets. It is composed of a set of intelligent proxies (Envoy) deployed as sidecars. These proxies mediate and control all network communication between microservices. They also collect and report telemetry on all mesh traffic
- Network topology refers to the way data flows in a network. The control plane establishes and changes network topology
- The telemetry is generated for workloads within a mesh. It collects network traffic data from devices on your network so that the data can be analyzed
- Istio uses an extended version of the Envoy proxy
- Envoy is a high-performance proxy developed in C++ to mediate all inbound and outbound traffic for all services in the service mesh
- Envoy proxies are the only Istio components that interact with data plane traffic. It keeps detailed statistics about network traffic. Envoy’s statistics cover the traffic for a particular Envoy instance

- A contol plane consists of

| Pilot  | Mixer | Citadel | Galley |
| --- | --- | --- | --- |
| Service discovery of destinations  | Collect metrics to Prometheus | Certificate authority | Istio management validation |
| Traffic features | Access control | Security | - |
| Blue/Green, Canary  | Receive telemetry from Envoy | Rbac | - |
| Timeout  | - | K8s access control | - |
| Retry  | - | - | - |
| Istio-proxy  | - | - | - |

- A data plane consists of
  - Ingress traffic
  - Service A
  - Proxy A
  - Mesh traffic
  - Proxy B
  - Service B
  - ...
  - Egress traffic

- A contol plane talks to data plan by Discovery Configuration Certificates

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
az group create --name $RESOURCE_GROUP --location $LOCATION
az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku $ACR_SKU
az acr login --name $ACR_NAME
```

## Step 2 create container registry

```bash
source .env
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

```bash
source .env

# create aks cluster
az aks create \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_NAME \
    --node-count 2 \
    --generate-ssh-keys \
    --attach-acr $ACR_NAME

# Configure kubectl to connect to aks
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_NAME
```

Check nodes

```bash
kubectl get nodes
```

## Step 5 Run app in aks

Assume env variable $ARC_SERVER contains the arc server url. This is used in azure-vote-all-in-one-redis.yaml.

The resources will be created in the order they appear in the file. Therefore, it's best to specify the service first, since that will ensure the scheduler can spread the pods associated with the service as they are created by the controller(s), such as Deployment.

Publish app

```bash
kubectl apply -f azure-vote-service.yaml \
              -f azure-vote-deploy.yaml
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
az aks show --resource-group $RESOURCE_GROUP --name $AKS_NAME --query kubernetesVersion --output table
```

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
```

```bash
kubectl autoscale deployment azure-vote-front --cpu-percent=50 --min=3 --max=10
```

```bash
kubectl apply -f azure-vote-hpa.yaml
```

Check hpa

```bash
kubectl get hpa
```

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
