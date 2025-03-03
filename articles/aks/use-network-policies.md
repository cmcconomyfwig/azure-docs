---
title: Secure pod traffic with network policy
titleSuffix: Azure Kubernetes Service
description: Learn how to secure traffic that flows in and out of pods by using Kubernetes network policies in Azure Kubernetes Service (AKS)
services: container-service
ms.topic: article
ms.date: 03/29/2022

---

# Secure traffic between pods using network policies in Azure Kubernetes Service (AKS)

When you run modern, microservices-based applications in Kubernetes, you often want to control which components can communicate with each other. The principle of least privilege should be applied to how traffic can flow between pods in an Azure Kubernetes Service (AKS) cluster. Let's say you likely want to block traffic directly to back-end applications. The *Network Policy* feature in Kubernetes lets you define rules for ingress and egress traffic between pods in a cluster.

This article shows you how to install the network policy engine and create Kubernetes network policies to control the flow of traffic between pods in AKS. Network policy should only be used for Linux-based nodes and pods in AKS.

## Before you begin

You need the Azure CLI version 2.0.61 or later installed and configured. Run `az --version` to find the version. If you need to install or upgrade, see [Install Azure CLI][install-azure-cli].

## Overview of network policy

All pods in an AKS cluster can send and receive traffic without limitations, by default. To improve security, you can define rules that control the flow of traffic. Back-end applications are often only exposed to required front-end services, for example. Or, database components are only accessible to the application tiers that connect to them.

Network Policy is a Kubernetes specification that defines access policies for communication between Pods. Using Network Policies, you define an ordered set of rules to send and receive traffic and apply them to a collection of pods that match one or more label selectors.

These network policy rules are defined as YAML manifests. Network policies can be included as part of a wider manifest that also creates a deployment or service.

### Network policy options in AKS

Azure provides two ways to implement network policy. You choose a network policy option when you create an AKS cluster. The policy option can't be changed after the cluster is created:

* Azure's own implementation, called *Azure Network Policies*.
* *Calico Network Policies*, an open-source network and network security solution founded by [Tigera][tigera].

Both implementations use Linux *IPTables* to enforce the specified policies. Policies are translated into sets of allowed and disallowed IP pairs. These pairs are then programmed as IPTable filter rules.

### Differences between Azure and Calico policies and their capabilities

| Capability                               | Azure                      | Calico                      |
|------------------------------------------|----------------------------|-----------------------------|
| Supported platforms                      | Linux                      | Linux, Windows Server 2019 (preview)  |
| Supported networking options             | Azure CNI                  | Azure CNI (Windows Server 2019 and Linux) and kubenet (Linux)  |
| Compliance with Kubernetes specification | All policy types supported |  All policy types supported |
| Additional features                      | None                       | Extended policy model consisting of Global Network Policy, Global Network Set, and Host Endpoint. For more information on using the `calicoctl` CLI to manage these extended features, see [calicoctl user reference][calicoctl]. |
| Support                                  | Supported by Azure support and Engineering team | Calico community support. For more information on additional paid support, see [Project Calico support options][calico-support]. |
| Logging                                  | Rules added / deleted in IPTables are logged on every host under */var/log/azure-npm.log* | For more information, see [Calico component logs][calico-logs] |

## Create an AKS cluster and enable network policy

To see network policies in action, let's create and then expand on a policy that defines traffic flow:

* Deny all traffic to pod.
* Allow traffic based on pod labels.
* Allow traffic based on namespace.

First, let's create an AKS cluster that supports network policy.

> [!IMPORTANT]
>
> The network policy feature can only be enabled when the cluster is created. You can't enable network policy on an existing AKS cluster.

To use Azure Network Policy, you must use the [Azure CNI plug-in][azure-cni] and define your own virtual network and subnets. For more detailed information on how to plan out the required subnet ranges, see [configure advanced networking][use-advanced-networking]. Calico Network Policy could be used with either this same Azure CNI plug-in or with the Kubenet CNI plug-in.

The following example script:

* Creates a virtual network and subnet.
* Creates an Azure Active Directory (Azure AD) service principal for use with the AKS cluster.
* Assigns *Contributor* permissions for the AKS cluster service principal on the virtual network.
* Creates an AKS cluster in the defined virtual network and enables network policy.
    * The _Azure Network_ policy option is used. To use Calico as the network policy option instead, use the `--network-policy calico` parameter. Note: Calico could be used with either `--network-plugin azure` or `--network-plugin kubenet`.

Note that instead of using a service principal, you can use a managed identity for permissions. For more information, see [Use managed identities](use-managed-identity.md).

Provide your own secure *SP_PASSWORD*. You can replace the *RESOURCE_GROUP_NAME* and *CLUSTER_NAME* variables:

```azurecli-interactive
RESOURCE_GROUP_NAME=myResourceGroup-NP
CLUSTER_NAME=myAKSCluster
LOCATION=canadaeast

# Create a resource group
az group create --name $RESOURCE_GROUP_NAME --location $LOCATION

# Create a virtual network and subnet
az network vnet create \
    --resource-group $RESOURCE_GROUP_NAME \
    --name myVnet \
    --address-prefixes 10.0.0.0/8 \
    --subnet-name myAKSSubnet \
    --subnet-prefix 10.240.0.0/16

# Create a service principal and read in the application ID
SP=$(az ad sp create-for-rbac --output json)
SP_ID=$(echo $SP | jq -r .appId)
SP_PASSWORD=$(echo $SP | jq -r .password)

# Wait 15 seconds to make sure that service principal has propagated
echo "Waiting for service principal to propagate..."
sleep 15

# Get the virtual network resource ID
VNET_ID=$(az network vnet show --resource-group $RESOURCE_GROUP_NAME --name myVnet --query id -o tsv)

# Assign the service principal Contributor permissions to the virtual network resource
az role assignment create --assignee $SP_ID --scope $VNET_ID --role Contributor

# Get the virtual network subnet resource ID
SUBNET_ID=$(az network vnet subnet show --resource-group $RESOURCE_GROUP_NAME --vnet-name myVnet --name myAKSSubnet --query id -o tsv)
```

### Create an AKS cluster for Azure network policies

Create the AKS cluster and specify the virtual network, service principal information, and *azure* for the network plugin and network policy.

```azurecli
az aks create \
    --resource-group $RESOURCE_GROUP_NAME \
    --name $CLUSTER_NAME \
    --node-count 1 \
    --generate-ssh-keys \
    --service-cidr 10.0.0.0/16 \
    --dns-service-ip 10.0.0.10 \
    --docker-bridge-address 172.17.0.1/16 \
    --vnet-subnet-id $SUBNET_ID \
    --service-principal $SP_ID \
    --client-secret $SP_PASSWORD \
    --network-plugin azure \
    --network-policy azure
```

It takes a few minutes to create the cluster. When the cluster is ready, configure `kubectl` to connect to your Kubernetes cluster by using the [az aks get-credentials][az-aks-get-credentials] command. This command downloads credentials and configures the Kubernetes CLI to use them:

```azurecli-interactive
az aks get-credentials --resource-group $RESOURCE_GROUP_NAME --name $CLUSTER_NAME
```

### Create an AKS cluster for Calico network policies

Create the AKS cluster and specify the virtual network, service principal information, *azure* for the network plugin, and *calico* for the network policy. Using *calico* as the network policy enables Calico networking on both Linux and Windows node pools.

If you plan on adding Windows node pools to your cluster, include the `windows-admin-username` and `windows-admin-password` parameters with that meet the [Windows Server password requirements][windows-server-password]. To use Calico with Windows node pools, you also need to register the `Microsoft.ContainerService/EnableAKSWindowsCalico`.

Register the `EnableAKSWindowsCalico` feature flag using the [az feature register][az-feature-register] command as shown in the following example:

```azurecli-interactive
az feature register --namespace "Microsoft.ContainerService" --name "EnableAKSWindowsCalico"
```

 You can check on the registration status using the [az feature list][az-feature-list] command:

```azurecli-interactive
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/EnableAKSWindowsCalico')].{Name:name,State:properties.state}"
```

When ready, refresh the registration of the *Microsoft.ContainerService* resource provider using the [az provider register][az-provider-register] command:

```azurecli-interactive
az provider register --namespace Microsoft.ContainerService
```

> [!IMPORTANT]
> At this time, using Calico network policies with Windows nodes is available on new clusters using Kubernetes version 1.20 or later with Calico 3.17.2 and requires using Azure CNI networking. Windows nodes on AKS clusters with Calico enabled also have [Direct Server Return (DSR)][dsr] enabled by default.
>
> For clusters with only Linux node pools running Kubernetes 1.20 with earlier versions of Calico, the Calico version will automatically be upgraded to 3.17.2.

Calico networking policies with Windows nodes is currently in preview.

[!INCLUDE [preview features callout](./includes/preview/preview-callout.md)]

Create a username to use as administrator credentials for your Windows Server containers on your cluster. The following commands prompt you for a username and set it WINDOWS_USERNAME for use in a later command (remember that the commands in this article are entered into a BASH shell).

```azurecli-interactive
echo "Please enter the username to use as administrator credentials for Windows Server containers on your cluster: " && read WINDOWS_USERNAME
```

```azurecli
az aks create \
    --resource-group $RESOURCE_GROUP_NAME \
    --name $CLUSTER_NAME \
    --node-count 1 \
    --generate-ssh-keys \
    --service-cidr 10.0.0.0/16 \
    --dns-service-ip 10.0.0.10 \
    --docker-bridge-address 172.17.0.1/16 \
    --vnet-subnet-id $SUBNET_ID \
    --service-principal $SP_ID \
    --client-secret $SP_PASSWORD \
    --windows-admin-username $WINDOWS_USERNAME \
    --vm-set-type VirtualMachineScaleSets \
    --kubernetes-version 1.20.2 \
    --network-plugin azure \
    --network-policy calico
```

It takes a few minutes to create the cluster. By default, your cluster is created with only a Linux node pool. If you would like to use Windows node pools, you can add one. For example:

```azurecli
az aks nodepool add \
    --resource-group $RESOURCE_GROUP_NAME \
    --cluster-name $CLUSTER_NAME \
    --os-type Windows \
    --name npwin \
    --node-count 1
```

When the cluster is ready, configure `kubectl` to connect to your Kubernetes cluster by using the [az aks get-credentials][az-aks-get-credentials] command. This command downloads credentials and configures the Kubernetes CLI to use them:

```azurecli-interactive
az aks get-credentials --resource-group $RESOURCE_GROUP_NAME --name $CLUSTER_NAME
```

## Deny all inbound traffic to a pod

Before you define rules to allow specific network traffic, first create a network policy to deny all traffic. This policy gives you a starting point to begin to create an allowlist for only the desired traffic. You can also clearly see that traffic is dropped when the network policy is applied.

For the sample application environment and traffic rules, let's first create a namespace called *development* to run the example pods:

```console
kubectl create namespace development
kubectl label namespace/development purpose=development
```

Create an example back-end pod that runs NGINX. This back-end pod can be used to simulate a sample back-end web-based application. Create this pod in the *development* namespace, and open port *80* to serve web traffic. Label the pod with *app=webapp,role=backend* so that we can target it with a network policy in the next section:

```console
kubectl run backend --image=mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine --labels app=webapp,role=backend --namespace development --expose --port 80
```

Create another pod and attach a terminal session to test that you can successfully reach the default NGINX webpage:

```console
kubectl run --rm -it --image=mcr.microsoft.com/dotnet/runtime-deps:6.0 network-policy --namespace development
```

Install `wget`:

```console
apt-get update && apt-get install -y wget
```

At the shell prompt, use `wget` to confirm that you can access the default NGINX webpage:

```console
wget -qO- http://backend
```

The following sample output shows that the default NGINX webpage returned:

```output
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
[...]
```

Exit out of the attached terminal session. The test pod is automatically deleted.

```console
exit
```

### Create and apply a network policy

Now that you've confirmed you can use the basic NGINX webpage on the sample back-end pod, create a network policy to deny all traffic. Create a file named `backend-policy.yaml` and paste the following YAML manifest. This manifest uses a *podSelector* to attach the policy to pods that have the *app:webapp,role:backend* label, like your sample NGINX pod. No rules are defined under *ingress*, so all inbound traffic to the pod is denied:

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: backend-policy
  namespace: development
spec:
  podSelector:
    matchLabels:
      app: webapp
      role: backend
  ingress: []
```

Go to [https://shell.azure.com](https://shell.azure.com) to open Azure Cloud Shell in your browser.

Apply the network policy by using the [kubectl apply][kubectl-apply] command and specify the name of your YAML manifest:

```console
kubectl apply -f backend-policy.yaml
```

### Test the network policy

Let's see if you can use the NGINX webpage on the back-end pod again. Create another test pod and attach a terminal session:

```console
kubectl run --rm -it --image=mcr.microsoft.com/dotnet/runtime-deps:6.0 network-policy --namespace development
```

Install `wget`:

```console
apt-get update && apt-get install -y wget
```

At the shell prompt, use `wget` to see if you can access the default NGINX webpage. This time, set a timeout value to *2* seconds. The network policy now blocks all inbound traffic, so the page can't be loaded, as shown in the following example:

```console
wget -O- --timeout=2 --tries=1 http://backend
```

```output
wget: download timed out
```

Exit out of the attached terminal session. The test pod is automatically deleted.

```console
exit
```

## Allow inbound traffic based on a pod label

In the previous section, a back-end NGINX pod was scheduled, and a network policy was created to deny all traffic. Let's create a front-end pod and update the network policy to allow traffic from front-end pods.

Update the network policy to allow traffic from pods with the labels *app:webapp,role:frontend* and in any namespace. Edit the previous *backend-policy.yaml* file, and add *matchLabels* ingress rules so that your manifest looks like the following example:

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: backend-policy
  namespace: development
spec:
  podSelector:
    matchLabels:
      app: webapp
      role: backend
  ingress:
  - from:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          app: webapp
          role: frontend
```

> [!NOTE]
> This network policy uses a *namespaceSelector* and a *podSelector* element for the ingress rule. The YAML syntax is important for the ingress rules to be additive. In this example, both elements must match for the ingress rule to be applied. Kubernetes versions prior to *1.12* might not interpret these elements correctly and restrict the network traffic as you expect. For more about this behavior, see [Behavior of to and from selectors][policy-rules].

Apply the updated network policy by using the [kubectl apply][kubectl-apply] command and specify the name of your YAML manifest:

```console
kubectl apply -f backend-policy.yaml
```

Schedule a pod that is labeled as *app=webapp,role=frontend* and attach a terminal session:

```console
kubectl run --rm -it frontend --image=mcr.microsoft.com/dotnet/runtime-deps:6.0 --labels app=webapp,role=frontend --namespace development
```

Install `wget`:

```console
apt-get update && apt-get install -y wget
```

At the shell prompt, use `wget` to see if you can access the default NGINX webpage:

```console
wget -qO- http://backend
```

Because the ingress rule allows traffic with pods that have the labels *app: webapp,role: frontend*, the traffic from the front-end pod is allowed. The following example output shows the default NGINX webpage returned:

```output
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
[...]
```

Exit out of the attached terminal session. The pod is automatically deleted.

```console
exit
```

### Test a pod without a matching label

The network policy allows traffic from pods labeled *app: webapp,role: frontend*, but should deny all other traffic. Let's test to see whether another pod without those labels can access the back-end NGINX pod. Create another test pod and attach a terminal session:

```console
kubectl run --rm -it --image=mcr.microsoft.com/dotnet/runtime-deps:6.0 network-policy --namespace development
```

Install `wget`:

```console
apt-get update && apt-get install -y wget
```

At the shell prompt, use `wget` to see if you can access the default NGINX webpage. The network policy blocks the inbound traffic, so the page can't be loaded, as shown in the following example:

```console
wget -O- --timeout=2 --tries=1 http://backend
```

```output
wget: download timed out
```

Exit out of the attached terminal session. The test pod is automatically deleted.

```console
exit
```

## Allow traffic only from within a defined namespace

In the previous examples, you created a network policy that denied all traffic, and then updated the policy to allow traffic from pods with a specific label. Another common need is to limit traffic to only within a given namespace. If the previous examples were for traffic in a *development* namespace, create a network policy that prevents traffic from another namespace, such as *production*, from reaching the pods.

First, create a new namespace to simulate a production namespace:

```console
kubectl create namespace production
kubectl label namespace/production purpose=production
```

Schedule a test pod in the *production* namespace that is labeled as *app=webapp,role=frontend*. Attach a terminal session:

```console
kubectl run --rm -it frontend --image=mcr.microsoft.com/dotnet/runtime-deps:6.0 --labels app=webapp,role=frontend --namespace production
```

Install `wget`:

```console
apt-get update && apt-get install -y wget
```

At the shell prompt, use `wget` to confirm that you can access the default NGINX webpage:

```console
wget -qO- http://backend.development
```

Because the labels for the pod match what is currently permitted in the network policy, the traffic is allowed. The network policy doesn't look at the namespaces, only the pod labels. The following example output shows the default NGINX webpage returned:

```output
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
[...]
```

Exit out of the attached terminal session. The test pod is automatically deleted.

```console
exit
```

### Update the network policy

Let's update the ingress rule *namespaceSelector* section to only allow traffic from within the *development* namespace. Edit the *backend-policy.yaml* manifest file as shown in the following example:

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: backend-policy
  namespace: development
spec:
  podSelector:
    matchLabels:
      app: webapp
      role: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          purpose: development
      podSelector:
        matchLabels:
          app: webapp
          role: frontend
```

In more complex examples, you could define multiple ingress rules, like a *namespaceSelector* and then a *podSelector*.

Apply the updated network policy by using the [kubectl apply][kubectl-apply] command and specify the name of your YAML manifest:

```console
kubectl apply -f backend-policy.yaml
```

### Test the updated network policy

Schedule another pod in the *production* namespace and attach a terminal session:

```console
kubectl run --rm -it frontend --image=mcr.microsoft.com/dotnet/runtime-deps:6.0 --labels app=webapp,role=frontend --namespace production
```

Install `wget`:

```console
apt-get update && apt-get install -y wget
```

At the shell prompt, use `wget` to see that the network policy now denies traffic:

```console
wget -O- --timeout=2 --tries=1 http://backend.development
```

```output
wget: download timed out
```

Exit out of the test pod:

```console
exit
```

With traffic denied from the *production* namespace, schedule a test pod back in the *development* namespace and attach a terminal session:

```console
kubectl run --rm -it frontend --image=mcr.microsoft.com/dotnet/runtime-deps:6.0 --labels app=webapp,role=frontend --namespace development
```

Install `wget`:

```console
apt-get update && apt-get install -y wget
```

At the shell prompt, use `wget` to see that the network policy allows the traffic:

```console
wget -qO- http://backend
```

Traffic is allowed because the pod is scheduled in the namespace that matches what's permitted in the network policy. The following sample output shows the default NGINX webpage returned:

```output
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
[...]
```

Exit out of the attached terminal session. The test pod is automatically deleted.

```console
exit
```

## Clean up resources

In this article, we created two namespaces and applied a network policy. To clean up these resources, use the [kubectl delete][kubectl-delete] command and specify the resource names:

```console
kubectl delete namespace production
kubectl delete namespace development
```

## Next steps

For more about network resources, see [Network concepts for applications in Azure Kubernetes Service (AKS)][concepts-network].

To learn more about policies, see [Kubernetes network policies][kubernetes-network-policies].

<!-- LINKS - external -->
[kubectl-apply]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply
[kubectl-delete]: https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete
[kubernetes-network-policies]: https://kubernetes.io/docs/concepts/services-networking/network-policies/
[azure-cni]: https://github.com/Azure/azure-container-networking/blob/master/docs/cni.md
[policy-rules]: https://kubernetes.io/docs/concepts/services-networking/network-policies/#behavior-of-to-and-from-selectors
[aks-github]: https://github.com/azure/aks/issues
[tigera]: https://www.tigera.io/
[calicoctl]: https://docs.projectcalico.org/reference/calicoctl/
[calico-support]: https://www.tigera.io/tigera-products/calico/
[calico-logs]: https://docs.projectcalico.org/maintenance/troubleshoot/component-logs
[calico-aks-cleanup]: https://github.com/Azure/aks-engine/blob/master/docs/topics/calico-3.3.1-cleanup-after-upgrade.yaml

<!-- LINKS - internal -->
[install-azure-cli]: /cli/azure/install-azure-cli
[use-advanced-networking]: configure-azure-cni.md
[az-aks-get-credentials]: /cli/azure/aks#az_aks_get_credentials
[concepts-network]: concepts-network.md
[az-feature-register]: /cli/azure/feature#az_feature_register
[az-feature-list]: /cli/azure/feature#az_feature_list
[az-provider-register]: /cli/azure/provider#az_provider_register
[windows-server-password]: /windows/security/threat-protection/security-policy-settings/password-must-meet-complexity-requirements#reference
[az-extension-add]: /cli/azure/extension#az_extension_add
[az-extension-update]: /cli/azure/extension#az_extension_update
[dsr]: ../load-balancer/load-balancer-multivip-overview.md#rule-type-2-backend-port-reuse-by-using-floating-ip
