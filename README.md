# Scaling AKS Nodes: Leveraging Cluster Autoscaler, Karpenter, and Node Autoprovision

## Meetup Details

- Click [here](https://www.meetup.com/de-DE/berlin-kubernetes-meetup/events/298300445/) to view the meetup details

## Slides

- Find the slides [here](https://www.slideshare.net/PhilipWelz/scaling-aks-nodes-leveraging-cluster-autoscaler-karpenter-and-node-autoprovision)

## Demo

### Cluster-Autoscaler

Create an AKS cluster with cluster-autoscaler enabled

#### Prerequisites

```bash
# Create Resource Group
RG_CA=rg-ca-demo
az group create \
  --name $RG_CA \
  --location northeurope

# Create AKS with cluster-autoscaler enabled
AKS_CA=aks-ca-demo
az aks create \
  --name $AKS_CA \
  --resource-group $RG_CA \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10

# Get AKS credentials
az aks get-credentials --resource-group $RG_CA --name $AKS_CA
```

#### Demo

```bash
kubectl get nodes --show-labels | grep "node.kubernetes.io/instance-type"
kubectl get nodes --show-labels | grep "kubernetes.azure.com/nodepool-type"
kubectl get configmap -n kube-system cluster-autoscaler-status -o yaml
# open second terminal and watch cluster-autoscaler events
kubectl get events -A --field-selector source=cluster-autoscaler -w
kubectl create ns inflate
kubectl -n inflate apply -f ./demo/inflate.yaml
# switch back to terminal 1 and scale deployment
kubectl -n inflate scale deployment/inflate --replicas=10
# explore scheduler events
kubectl get events -n inflate \
 --sort-by=.metadata.creationTimestamp \
 --field-selector source=default-scheduler
kubectl -n inflate get pods -o wide
kubectl -n inflate scale deployment/inflate --replicas=11
kubectl -n inflate get pods -o wide
```

#### Cleanup

```bash
az aks delete --resource-group $RG_CA --name $AKS_CA --yes \
  && az group delete --name $RG_CA --yes
```

### Node-Autoprovision

Create an AKS cluster with node-autoprovision enabled (preview)

#### Prerequisites

```bash
# Install the aks-preview CLI extension
az extension add --name aks-preview

# Optional: Update the preview extension to make sure you have the latest version installed
az extension update --name aks-preview

# Register the NodeAutoProvisioningPreview feature flag
az feature register --namespace "Microsoft.ContainerService" --name "NodeAutoProvisioningPreview"

# Verify the registration status using the az feature show command.
az feature show --namespace "Microsoft.ContainerService" --name "NodeAutoProvisioningPreview"

# When the status reflects Registered, refresh the registration of the Microsoft.ContainerService resource provider using the az provider register command.
az provider register --namespace Microsoft.ContainerService

# Create AKS RG
RG_NAP=rg-nap-demo
az group create \
  --name $RG_NAP \
  --location northeurope

# Create AKS with node-autoprovision enabled
AKS_NAP=aks-nap-demo
az aks create \
  --name $AKS_NAP \
  --resource-group $RG_NAP \
  --node-provisioning-mode Auto \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --network-dataplane cilium

# Get AKS credentials
az aks get-credentials --resource-group $RG_NAP --name $AKS_NAP
```

#### Demo

```bash
kubectl get nodes
kubectl get nodes --show-labels | grep "node.kubernetes.io/instance-type"
kubectl get nodes --show-labels | grep "kubernetes.azure.com/nodepool-type"
# open new terminal and watch karpernter events
kubectl get events -A --field-selector source=karpenter -w
# switch back to terminal 1 and scale deployment
kubectl create ns inflate
kubectl -n inflate apply -f ./demo/inflate.yaml
kubectl -n inflate scale deployment/inflate --replicas=10
# explore karepenter resources
kubectl api-resources
kubectl get nodepools
kubectl get nodepools default -oyaml | yq
kubectl get nodepools system-surge -oyaml | yq
kubectl get aksnodeclasses
kubectl get aksnodeclasses default -oyaml | yq
kubectl get aksnodeclasses system-surge -oyaml | yq
kubectl -n inflate get pods -o wide
kubectl get nodeclaims
kubectl get nodeclaims default- -oyaml | yq
# play around with scaling
kubectl -n inflate scale deployment/inflate --replicas=15
kubectl get nodeclaims
kubectl -n inflate scale deployment/inflate --replicas=12
kubectl get nodeclaims
kubectl -n inflate scale deployment/inflate --replicas=4
kubectl get nodeclaims
```

#### Cleanup

```bash
az aks delete --resource-group $RG_NAP --name $AKS_NAP --yes \
  && az group delete --name $RG_NAP --yes
```

## Useful Links

- [Azure Cluster Autoscaler docs](https://learn.microsoft.com/en-us/azure/aks/cluster-autoscaler?tabs=azure-cli)
- [Azure Node autoprovision docs](https://learn.microsoft.com/en-us/azure/aks/node-autoprovision?tabs=azure-cli)
- [Azure Karpenter GitHub](https://github.com/Azure/karpenter)
- [Exploring Azure Kubernetes Service’s Node Autoprovision](https://pixelrobots.co.uk/2023/12/exploring-azure-kubernetes-services-node-autoprovision-a-deep-dive-into-the-latest-public-preview-feature/)
