# Azure Kubernetes Service (AKS) GitOps cluster

## Create an AKS cluster

```bash

# set env vars
export RESOURCE_GROUP="aks"
export CLUSTER_NAME="miqa"
export FLUX_CONFIG_NAME="platform"
export FLUX_NAMESPACE="flux-system"
export GIT_URL="https://github.com/bartr/gitops-platform"
export GIT_BRANCH="bartr"

```

```bash

az extension add -n connectedk8s
az extension add -n k8s-extension

az extension update -n k8s-configuration
az extension update -n k8s-extension

az provider register -n Microsoft.Kubernetes
az provider register -n Microsoft.KubernetesConfiguration
az provider register -n Microsoft.ExtendedLocation

az group create -n $RESOURCE_GROUP -l centralus

az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --node-count 1 \
  --node-vm-size Standard_D4S_V6 \
  --enable-managed-identity \
  --enable-addons monitoring \
  --network-plugin azure \
  --max-pods 250 \
  --generate-ssh-keys

# merge AKS credentials
az aks get-credentials -g $RESOURCE_GROUP -n $CLUSTER_NAME

# connect the AKS cluster to ARC
az connectedk8s connect -g $RESOURCE_GROUP -n $CLUSTER_NAME

# export AAD_ENTITY_ID=$(az ad group show --group <group-name> --query id -o tsv)

export AAD_ENTITY_ID=$(az ad signed-in-user show --query userPrincipalName -o tsv) && echo $AAD_ENTITY_ID

kubectl create clusterrolebinding arc-aad-binding --clusterrole cluster-admin --user=$AAD_ENTITY_ID

az role assignment create --role "Azure Arc Kubernetes Viewer" --assignee $AAD_ENTITY_ID --scope "/subscriptions/ca9b7a74-767e-427c-82f4-804e6f747c99/resourceGroups/aks/providers/Microsoft.Kubernetes/connectedClusters/miqa"

az role assignment create --role "Azure Arc Enabled Kubernetes Cluster User Role" --assignee $AAD_ENTITY_ID --scope "/subscriptions/ca9b7a74-767e-427c-82f4-804e6f747c99/resourceGroups/aks/providers/Microsoft.Kubernetes/connectedClusters/miqa"

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

kubectl get secret arc-user-secret -o jsonpath='{$.data.token}' | base64 -d | sed 's/$/\n/g'

```

```bash

az k8s-extension create \
  --resource-group "$RESOURCE_GROUP" \
  --cluster-name "$CLUSTER_NAME" \
  --cluster-type connectedClusters \
  --name flux \
  --extension-type microsoft.flux

az k8s-configuration flux create \
  --resource-group "$RESOURCE_GROUP" \
  --cluster-name "$CLUSTER_NAME" \
  --cluster-type connectedClusters \
  --name "$FLUX_CONFIG_NAME" \
  --scope cluster \
  --namespace "$FLUX_NAMESPACE" \
  --url "$GIT_URL" \
  --branch "$GIT_BRANCH" \
  --kustomization name=listeners path=./platform/$CLUSTER_NAME/listeners prune=true

az k8s-configuration flux create \
  --resource-group "$RESOURCE_GROUP" \
  --cluster-name "$CLUSTER_NAME" \
  --cluster-type connectedClusters \
  --name apps \
  --scope cluster \
  --namespace "$FLUX_NAMESPACE" \
  --url "$GIT_URL" \
  --branch "$GIT_BRANCH" \
  --kustomization name=listeners path=./apps/$CLUSTER_NAME/listeners prune=true

```

## Contributions

Please see our [Contributor guide](./CONTRIBUTING.md).

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact <opencode@microsoft.com> with any additional questions or comments.

With :heart: from Microsoft Patterns & Practices, [Azure Architecture Center](https://aka.ms/architecture).
