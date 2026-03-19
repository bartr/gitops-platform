# Azure Kubernetes Service (AKS) GitOps cluster

## Agenda

Spike goals
- Create a GitOps `starting point`
  - Reuse as much of the WCNP learnings as possible
  - Reuse as much of the Domino's GitOps as possible
- Assume the Platform Team has limited K8s expertise
  - Keep it simple
- Current plan is for one app team to deploy one app to the cluster
  - It is expected to grow over time
  - We don't have all of the requirements yet

Demo ARC on wiqa cluster
- Propose some additional standards
- Azure JIT permissions
- Entra IDs "just work"
- Namespaces
- Services
- Workloads
- GitOps
  - Configurations (sources)
  - Kustomizations
  - Deploy a new version of `heartbeat` and observe

## Questions

- Is the ARC UI sufficient for the Platform Team with limited K8s experience?
- Does `az connectedk8s proxy` work with `private AKS`?
  - If so, can we simplify the architecture and remove bastion / jump boxes?

## Next Steps

- Deploy private AKS and test ARC proxy
- Deploy cert-manager, let's encrypt, and Envoy for north/south with TLS termination
- Merge with the TF process

## Installation

## Add Azure extensions and providers

```bash

# only run this one time
az extension add -n connectedk8s
az extension add -n k8s-extension
az extension update -n k8s-configuration
az extension update -n k8s-extension

az provider register -n Microsoft.Kubernetes
az provider register -n Microsoft.KubernetesConfiguration
az provider register -n Microsoft.ExtendedLocation

```

## Create an AKS cluster

```bash

# set env vars
export RESOURCE_GROUP="aks"
export CLUSTER_NAME="miqa"
export AAD_GROUP_ID=$(az ad group show --group "MIQA Tenant Administrators" --query id -o tsv | tr -d '\r')
export AAD_TENANT_ID=$(az account show --query tenantId -o tsv | tr -d '\r')

# create resource group
az group create -n $RESOURCE_GROUP -l centralus

# create AKS cluster
az aks create \
  --resource-group "$RESOURCE_GROUP" \
  --name "$CLUSTER_NAME" \
  --aad-admin-group-object-ids "$AAD_GROUP_ID" \
  --aad-tenant-id "$AAD_TENANT_ID" \
  --node-count 1 \
  --node-vm-size Standard_D4S_V6 \
  --enable-managed-identity \
  --enable-aad \
  --enable-azure-rbac \
  --enable-addons monitoring \
  --network-plugin azure \
  --max-pods 250 \
  --generate-ssh-keys

# merge AKS credentials
az aks get-credentials -g $RESOURCE_GROUP -n $CLUSTER_NAME

# connect the AKS cluster to ARC
az connectedk8s connect -g $RESOURCE_GROUP -n $CLUSTER_NAME

# create flux extension
az k8s-extension create \
  --resource-group "$RESOURCE_GROUP" \
  --cluster-name "$CLUSTER_NAME" \
  --cluster-type connectedClusters \
  --name flux \
  --extension-type microsoft.flux

# create platform GitOps config
az k8s-configuration flux create \
  --resource-group "$RESOURCE_GROUP" \
  --cluster-name "$CLUSTER_NAME" \
  --cluster-type connectedClusters \
  --name platform \
  --scope cluster \
  --namespace flux-system \
  --url https://github.com/bartr/gitops-platform \
  --branch bartr \
  --kustomization \
    name=cert-manager \
    path=./platform/$CLUSTER_NAME/cert-manager \
    sync-interval=1m \
    timeout=3m \
    prune=true \
    force=true \
  --kustomization \
    name=heartbeat \
    path=./platform/$CLUSTER_NAME/heartbeat \
    sync-interval=1m \
    timeout=3m \
    prune=true \
    force=true \
  --no-wait

# create apps GitOps config
az k8s-configuration flux create \
  --resource-group "$RESOURCE_GROUP" \
  --cluster-name "$CLUSTER_NAME" \
  --cluster-type connectedClusters \
  --name apps \
  --scope cluster \
  --namespace flux-system \
  --url https://github.com/bartr/gitops-apps \
  --branch bartr \
  --kustomization \
    name=timeclock \
    path=./apps/$CLUSTER_NAME/timeclock \
    sync-interval=1m \
    timeout=3m \
    prune=true \
    force=true \
  --no-wait

```

```bash
# not needed when using AAD
# create a service account and secret
kubectl create serviceaccount arc-user -n default
kubectl create clusterrolebinding arc-user-binding --clusterrole cluster-admin --serviceaccount default:arc-user

kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: arc-user-secret
  annotations:
    kubernetes.io/service-account.name: arc-user
type: kubernetes.io/service-account-token
EOF

# get the ARC secret
kubectl get secret arc-user-secret -o jsonpath='{$.data.token}' | base64 -d | sed 's/$/\n/g'

```
