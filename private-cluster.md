# AKS GitOps Private Cluster Spike

## Installation

## Create the shared identity, network, Bastion host, and jump box

```bash

# run these commands from your local workstation

# save the local environment variables so they can be reused later
export GOA_CLUSTER_NAME="miqa"
export GOA_LOCATION="centralus"
export GOA_ENV_FILE="$HOME/goa.env"

# Save the environment variables
cat > "$GOA_ENV_FILE" <<EOF
export GOA_CLUSTER_NAME="${GOA_CLUSTER_NAME}"
export GOA_LOCATION="${GOA_LOCATION}"
export GOA_IDENTITY_RESOURCE_GROUP="rg-\${GOA_CLUSTER_NAME}-identity"
export GOA_NETWORK_RESOURCE_GROUP="rg-\${GOA_CLUSTER_NAME}-network"
export GOA_AKS_RESOURCE_GROUP="rg-\${GOA_CLUSTER_NAME}-aks"
export GOA_ADMIN_RESOURCE_GROUP="rg-\${GOA_CLUSTER_NAME}-admin"
export GOA_VNET_NAME="\${GOA_CLUSTER_NAME}-vnet"
export GOA_AKS_SUBNET_NAME="aks-subnet"
export GOA_ADMIN_SUBNET_NAME="admin-subnet"
export GOA_BASTION_SUBNET_NAME="AzureBastionSubnet"
export GOA_BASTION_NAME="\${GOA_CLUSTER_NAME}-bastion"
export GOA_BASTION_PIP_NAME="\${GOA_CLUSTER_NAME}-bastion-pip"
export GOA_BASTION_TUNNEL_PORT="50022"
export GOA_UAMI_NAME="\${GOA_CLUSTER_NAME}-admin-uami"
export GOA_VM_NAME="\${GOA_CLUSTER_NAME}-admin"
export GOA_ADMIN_USERNAME="azureuser"
EOF

source "$GOA_ENV_FILE"

export GOA_AAD_GROUP_ID=$(az ad group show --group "MIQA Tenant Administrators" --query id -o tsv | tr -d '\r')
export GOA_AAD_TENANT_ID=$(az account show --query tenantId -o tsv | tr -d '\r')
export GOA_SUBSCRIPTION_ID=$(az account show --query id -o tsv | tr -d '\r')

cat >> "$GOA_ENV_FILE" <<EOF
export GOA_AAD_GROUP_ID="$GOA_AAD_GROUP_ID"
export GOA_AAD_TENANT_ID="$GOA_AAD_TENANT_ID"
export GOA_SUBSCRIPTION_ID="$GOA_SUBSCRIPTION_ID"
EOF

# create resource groups
az group create -n "$GOA_IDENTITY_RESOURCE_GROUP" -l "$GOA_LOCATION"

# create a reusable user-assigned managed identity for the admin VM
az identity create \
  --resource-group "$GOA_IDENTITY_RESOURCE_GROUP" \
  --location "$GOA_LOCATION" \
  --name "$GOA_UAMI_NAME"

export GOA_UAMI_RESOURCE_ID=$(az identity show \
  --resource-group "$GOA_IDENTITY_RESOURCE_GROUP" \
  --name "$GOA_UAMI_NAME" \
  --query id -o tsv | tr -d '\r')

export GOA_UAMI_CLIENT_ID=$(az identity show \
  --resource-group "$GOA_IDENTITY_RESOURCE_GROUP" \
  --name "$GOA_UAMI_NAME" \
  --query clientId -o tsv | tr -d '\r')

export GOA_UAMI_PRINCIPAL_ID=$(az identity show \
  --resource-group "$GOA_IDENTITY_RESOURCE_GROUP" \
  --name "$GOA_UAMI_NAME" \
  --query principalId -o tsv | tr -d '\r')

cat >> "$GOA_ENV_FILE" <<EOF
export GOA_UAMI_RESOURCE_ID="$GOA_UAMI_RESOURCE_ID"
export GOA_UAMI_CLIENT_ID="$GOA_UAMI_CLIENT_ID"
export GOA_UAMI_PRINCIPAL_ID="$GOA_UAMI_PRINCIPAL_ID"
EOF

az group create -n "$GOA_NETWORK_RESOURCE_GROUP" -l "$GOA_LOCATION"
az group create -n "$GOA_AKS_RESOURCE_GROUP" -l "$GOA_LOCATION"
az group create -n "$GOA_ADMIN_RESOURCE_GROUP" -l "$GOA_LOCATION"

# create VNet and subnets
az network vnet create \
  --resource-group "$GOA_NETWORK_RESOURCE_GROUP" \
  --location "$GOA_LOCATION" \
  --name "$GOA_VNET_NAME" \
  --address-prefixes 10.20.0.0/16 \
  --subnet-name "$GOA_AKS_SUBNET_NAME" \
  --subnet-prefixes 10.20.0.0/22

az network vnet subnet create \
  --resource-group "$GOA_NETWORK_RESOURCE_GROUP" \
  --vnet-name "$GOA_VNET_NAME" \
  --name "$GOA_ADMIN_SUBNET_NAME" \
  --address-prefixes 10.20.10.0/24

az network vnet subnet create \
  --resource-group "$GOA_NETWORK_RESOURCE_GROUP" \
  --vnet-name "$GOA_VNET_NAME" \
  --name "$GOA_BASTION_SUBNET_NAME" \
  --address-prefixes 10.20.254.0/26

export GOA_AKS_SUBNET_ID=$(az network vnet subnet show \
  --resource-group "$GOA_NETWORK_RESOURCE_GROUP" \
  --vnet-name "$GOA_VNET_NAME" \
  --name "$GOA_AKS_SUBNET_NAME" \
  --query id -o tsv | tr -d '\r')

export GOA_ADMIN_SUBNET_ID=$(az network vnet subnet show \
  --resource-group "$GOA_NETWORK_RESOURCE_GROUP" \
  --vnet-name "$GOA_VNET_NAME" \
  --name "$GOA_ADMIN_SUBNET_NAME" \
  --query id -o tsv | tr -d '\r')

cat >> "$GOA_ENV_FILE" <<EOF
export GOA_AKS_SUBNET_ID="$GOA_AKS_SUBNET_ID"
export GOA_ADMIN_SUBNET_ID="$GOA_ADMIN_SUBNET_ID"
EOF

# create the Bastion public IP and host
az network public-ip create \
  --resource-group "$GOA_NETWORK_RESOURCE_GROUP" \
  --location "$GOA_LOCATION" \
  --name "$GOA_BASTION_PIP_NAME" \
  --sku Standard

az network bastion create \
  --resource-group "$GOA_NETWORK_RESOURCE_GROUP" \
  --location "$GOA_LOCATION" \
  --name "$GOA_BASTION_NAME" \
  --sku Standard \
  --public-ip-address "$GOA_BASTION_PIP_NAME" \
  --vnet-name "$GOA_VNET_NAME"

# create a Linux admin VM in the same VNet as AKS without a public IP
az vm create \
  --resource-group "$GOA_ADMIN_RESOURCE_GROUP" \
  --location "$GOA_LOCATION" \
  --name "$GOA_VM_NAME" \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username "$GOA_ADMIN_USERNAME" \
  --generate-ssh-keys \
  --assign-identity "$GOA_UAMI_RESOURCE_ID" \
  --subnet "$GOA_ADMIN_SUBNET_ID" \
  --public-ip-address ""

export GOA_JUMP_BOX_ID=$(az vm show \
  --resource-group "$GOA_ADMIN_RESOURCE_GROUP" \
  --name "$GOA_VM_NAME" \
  --query id -o tsv | tr -d '\r')

cat >> "$GOA_ENV_FILE" <<EOF
export GOA_JUMP_BOX_ID="$GOA_JUMP_BOX_ID"
EOF

# grant the reusable identity the access it needs across the shared resource groups
# these role assignments only need to be created once per identity
az role assignment create \
  --assignee-object-id "$GOA_UAMI_PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role Reader \
  --scope $(az group show --name "$GOA_IDENTITY_RESOURCE_GROUP" --query id -o tsv)

az role assignment create \
  --assignee-object-id "$GOA_UAMI_PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Network Contributor" \
  --scope $(az group show --name "$GOA_NETWORK_RESOURCE_GROUP" --query id -o tsv)

az role assignment create \
  --assignee-object-id "$GOA_UAMI_PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role Contributor \
  --scope $(az group show --name "$GOA_AKS_RESOURCE_GROUP" --query id -o tsv)

az role assignment create \
  --assignee-object-id "$GOA_UAMI_PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "Azure Kubernetes Service RBAC Cluster Admin" \
  --scope $(az group show --name "$GOA_AKS_RESOURCE_GROUP" --query id -o tsv)

# start a native client tunnel from your local machine to the private jump box
# keep this running in its own local terminal window
az network bastion tunnel \
  --resource-group "$GOA_NETWORK_RESOURCE_GROUP" \
  --name "$GOA_BASTION_NAME" \
  --target-resource-id "$GOA_JUMP_BOX_ID" \
  --resource-port 22 \
  --port "$GOA_BASTION_TUNNEL_PORT"

# from a second local terminal, reload the saved variables before using scp or ssh
source ~/goa.env

# copy the env file to the jump box through the local tunnel
scp -P "$GOA_BASTION_TUNNEL_PORT" "$GOA_ENV_FILE" "$GOA_ADMIN_USERNAME@127.0.0.1:~/goa.env"

# configure the jump box to auto-load the copied env file for the current bootstrap shell
ssh -p "$GOA_BASTION_TUNNEL_PORT" "$GOA_ADMIN_USERNAME@127.0.0.1" <<EOF
source "$HOME/goa.env"
EOF

# then SSH to the private VM through the local tunnel
ssh -p "$GOA_BASTION_TUNNEL_PORT" "$GOA_ADMIN_USERNAME@127.0.0.1"

# in future local terminals, reload the saved variables with:
# source "$GOA_ENV_FILE"

```

## Prepare the jump box

```bash

# run these commands inside the Linux VM

# if you ran the bootstrap step above, the env file should already be loaded for this session
# if not, load it manually with:
# source "$HOME/goa.env"

export GOA_VM_ENV_FILE="$HOME/goa.env"

sudo apt-get update
sudo apt-get install -y zsh

# install oh-my-zsh without switching shells mid-script
if [ ! -d "$HOME/.oh-my-zsh" ]; then
  RUNZSH=no CHSH=no KEEP_ZSHRC=yes sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
fi

# auto-load goa.env in future zsh sessions
touch "$HOME/.zshrc"
grep -qxF 'test -f "$HOME/goa.env" && source "$HOME/goa.env"' "$HOME/.zshrc" || \
  echo 'test -f "$HOME/goa.env" && source "$HOME/goa.env"' >> "$HOME/.zshrc"

# make zsh the login shell for the jump-box user
chsh -s "$(command -v zsh)"

curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az extension add -n connectedk8s
az extension add -n k8s-extension
az extension update -n k8s-configuration
az extension update -n k8s-extension
az aks install-cli

# authenticate from the jump box with the user-assigned managed identity
export GOA_UAMI_CLIENT_ID=$(az identity show \
  --resource-group "$GOA_IDENTITY_RESOURCE_GROUP" \
  --name "$GOA_UAMI_NAME" \
  --query clientId -o tsv | tr -d '\r')

export GOA_UAMI_PRINCIPAL_ID=$(az identity show \
  --resource-group "$GOA_IDENTITY_RESOURCE_GROUP" \
  --name "$GOA_UAMI_NAME" \
  --query principalId -o tsv | tr -d '\r')

cat >> "$GOA_VM_ENV_FILE" <<EOF
export GOA_UAMI_CLIENT_ID="$GOA_UAMI_CLIENT_ID"
export GOA_UAMI_PRINCIPAL_ID="$GOA_UAMI_PRINCIPAL_ID"
EOF

az login --identity --username "$GOA_UAMI_CLIENT_ID"
az account set --subscription "$GOA_SUBSCRIPTION_ID"

# open a new shell when you want zsh and oh-my-zsh to take effect immediately
# exec zsh

```

## Create the private AKS cluster

```bash

# run these commands inside the Linux VM

# resolve the AKS subnet ID from the VM session
export GOA_AKS_SUBNET_ID=$(az network vnet subnet show \
  --resource-group "$GOA_NETWORK_RESOURCE_GROUP" \
  --vnet-name "$GOA_VNET_NAME" \
  --name "$GOA_AKS_SUBNET_NAME" \
  --query id -o tsv | tr -d '\r')

cat >> "$GOA_VM_ENV_FILE" <<EOF
export GOA_AKS_SUBNET_ID="$GOA_AKS_SUBNET_ID"
EOF

# create AKS cluster
az aks create \
  --resource-group "$GOA_AKS_RESOURCE_GROUP" \
  --name "$GOA_CLUSTER_NAME" \
  --location "$GOA_LOCATION" \
  --enable-managed-identity \
  --enable-aad \
  --aad-admin-group-object-ids "$GOA_AAD_GROUP_ID" \
  --aad-tenant-id "$GOA_AAD_TENANT_ID" \
  --enable-azure-rbac \
  --node-count 1 \
  --node-vm-size Standard_D4s_v6 \
  --enable-addons monitoring \
  --network-plugin azure \
  --vnet-subnet-id "$GOA_AKS_SUBNET_ID" \
  --max-pods 250 \
  --enable-private-cluster \
  --disable-public-fqdn \
  --private-dns-zone system \
  --generate-ssh-keys

# run the remaining commands from the jump box so they can reach the private API server

# merge AKS credentials
az aks get-credentials -g "$GOA_AKS_RESOURCE_GROUP" -n "$GOA_CLUSTER_NAME"

# verify the jump box can resolve and reach the private AKS API server
kubectl get nodes

# connect the AKS cluster to ARC
az connectedk8s connect -g "$GOA_AKS_RESOURCE_GROUP" -n "$GOA_CLUSTER_NAME"

# create flux extension
az k8s-extension create \
  --resource-group "$GOA_AKS_RESOURCE_GROUP" \
  --cluster-name "$GOA_CLUSTER_NAME" \
  --cluster-type connectedClusters \
  --name flux \
  --extension-type microsoft.flux

# create platform GitOps config
az k8s-configuration flux create \
  --resource-group "$GOA_AKS_RESOURCE_GROUP" \
  --cluster-name "$GOA_CLUSTER_NAME" \
  --cluster-type connectedClusters \
  --name platform \
  --scope cluster \
  --namespace flux-system \
  --url https://github.com/bartr/gitops-platform \
  --branch bartr \
  --kustomization \
    name=cert-manager \
    path=./platform/$GOA_CLUSTER_NAME/cert-manager \
    sync-interval=1m \
    timeout=3m \
    prune=true \
    force=true \
  --kustomization \
    name=heartbeat \
    path=./platform/$GOA_CLUSTER_NAME/heartbeat \
    sync-interval=1m \
    timeout=3m \
    prune=true \
    force=true \
  --no-wait

# create apps GitOps config
az k8s-configuration flux create \
  --resource-group "$GOA_AKS_RESOURCE_GROUP" \
  --cluster-name "$GOA_CLUSTER_NAME" \
  --cluster-type connectedClusters \
  --name apps \
  --scope cluster \
  --namespace flux-system \
  --url https://github.com/bartr/gitops-apps \
  --branch bartr \
  --kustomization \
    name=timeclock \
    path=./apps/$GOA_CLUSTER_NAME/timeclock \
    sync-interval=1m \
    timeout=3m \
    prune=true \
    force=true \
  --no-wait

```
