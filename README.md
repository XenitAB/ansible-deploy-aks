# ansible-deploy-aks

## Introduction
Ansible deployment of AKS

## Information
Two playbooks to deploy and configure Azure Kubernetes Service.

Includes the following in Azure:
* AKS Cluster (RBAC)
* ContainerInsights (disabled by default)
* DNS Zone (to be configured with external-dns)
* Azure Container Registry
* Not implemented: Azure AD Configuration

Includes the following in Kubernetes:
* Istio
* cert-manager
* external-dns
* goldpinger
* ark (velero)
* kubedb (disabled by default)
* datadog agent (and tracing)

## Configure the following files
* [deploy-aks/group_vars/all.yml](deploy-aks/group_vars/all.yml)
* [configure-aks/group_vars/all.yml](configure-aks/group_vars/all.yml)
* [configure-aks/roles/configure-aks/defaults/main.yml](configure-aks/roles/configure-aks/defaults/main.yml)

## How to run

### TODO: Create Azure AD Application (isn't implemented as of now)
Follow this guide: [Integrate Azure Active Directory with Azure Kubernetes Service](https://docs.microsoft.com/en-us/azure/aks/aad-integration)

### Generate service principal for Azure
```azcli
# Login to Azure
az login

# List subscriptions
az account list --output table

# Select subscription
az account set --subscription "<Subscription Name>"

# Valdiate that the correct subscription is selected
az account list --query "[?isDefault==\`true\`]" --output table

# Create variables with the subscription data
TenantID=$(az account list --query "[?isDefault==\`true\`].tenantId" --output tsv)
SubscriptionID=$(az account list --query "[?isDefault==\`true\`].id" --output tsv)

# Create Service Principal
# Store the password in a safe place and write down that it will expire in a year
# This will make the service principal contributor to the subscription
az ad sp create-for-rbac --name sp-aks
ClientID=$(az ad sp list --query "[?appDisplayName=='sp-aks'].appId" --output tsv)
```

### Ansible
#### Deploy AKS
```ansible
cd deploy-aks
ansible-playbook -i hosts deploy-aks.yml -e "ansible_python_interpreter=<python>" -e AZURE_CLIENT_ID="<ClientID>" -e AZURE_SECRET='"<Secret>"' -e AZURE_SUBSCRIPTION_ID="<SubscriptionID>" -e AZURE_TENANT="<TenantID>" --flush-cache
```

~~TODO: AAD Integration: (not included as of now)~~

~~ansible-playbook -i hosts-prd deploy-aks.yml -e "ansible_python_interpreter=<python>" -e AZURE_CLIENT_ID="<ClientID>" -e AZURE_SECRET='"<Secret>"' -e AZURE_SUBSCRIPTION_ID="<SubscriptionID>" -e AZURE_TENANT="<TenantID>" -e aksAADClientAppID="<AKS AAD Client App ID>" -e aksAADServerAppID="<AKS AAD Server App ID>" -e aksAADServerAppSecret='"<AKS AAD Server App Secret>"' -e aksAADTenantID="<AKS AAD Tenant ID>" --flush-cache~~


Manual steps:
* Configure the nameservers of the domain, pointing to the zone created in the resource group. Do this before running configure-aks.

#### Configure AKS
```ansible
cd configure-aks
ansible-playbook -i hosts configure-aks.yml -e "ansible_python_interpreter=<python>" -e AZURE_CLIENT_ID="<ClientID>" -e AZURE_SECRET='"<Secret>"' -e AZURE_SUBSCRIPTION_ID="<SubscriptionID>" -e AZURE_TENANT="<TenantID>" -e DATADOG_API_KEY='"<DatadogApiKey>"' --flush-cache
```

### Kubernetes
#### Goldpinger
```kubectl
kubectl -n goldpinger port-forward $(kubectl -n goldpinger get pod -l app=goldpinger -o jsonpath='{.items[0].metadata.name}') 8080:80
```