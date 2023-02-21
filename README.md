# essential-aks

https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-portal?tabs=azure-cli

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

https://aka.ms/ArcK8sOutboundURLs


## Step X Create an Azure Arc Private Link Scope


## Step X Local machine install az cli and extensions
connectedk8s and k8s-configuration, k8s-extension, customlocation CLI extensions


## Step X Local machine install cli, extensions

get a kubeconfig file https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/
