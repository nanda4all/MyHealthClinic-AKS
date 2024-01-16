# MyHealthClinic-AKS
ASPNetCore Project with SQL Server. Working with Azure DevOps CICD Pipeline on Azure Kubernetes Services
:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::

Steps
  1) Create the infrastructure using the below script file in Azure CLI ( Azure Cloud Shell from Azure portal)
  2) Install two Azure DevOps extensions and enable them in your organization
           i) Replace Token
          ii) Kubernetes extension
  3) Clone and create the repo in DevOps
  4) Make the changes in azure-pipelines.yml
  5) Azure Cloud Shell -> Login into the created AKS cluster by downloading cluster credentials, and Check the noded, pods, deploy, and services. Browse the LoadBalancer External IP

The application should be running. 
Username: user and Password: P2ssw0rd@1


**The following resources will be provisioned:**
------------------------------------------------------
    A Resource Group
    An AKS Kubernetes Cluster
    An Image container registry
    A SQL Server
    A SQL Database
=========================================================================================================================
**Below are two script files to create and destroy the infrastructure**
=======================================================================
**1) create_infra.sh**
=======================================================================
#!/bin/bash
REGION="westus"
RGP="day11-demo-rg"
CLUSTER_NAME="day11-demo-cluster"
ACR_NAME="day11demoacr"
SQLSERVER="day11-demo-sqlserver"
DB="mhcdb"

#Create Resource group
az group create --name $RGP --location $REGION

#Deploy AKS
az aks create --resource-group $RGP --name $CLUSTER_NAME --enable-addons monitoring --generate-ssh-keys --location $REGION

#Deploy ACR
az acr create --resource-group $RGP --name $ACR_NAME --sku Standard --location $REGION

#Authenticate with ACR to AKS
az aks update -n $CLUSTER_NAME -g $RGP --attach-acr $ACR_NAME

#Create SQL Server and DB
az sql server create -l $REGION -g $RGP -n $SQLSERVER -u sqladmin -p P2ssw0rd1234

az sql db create -g $RGP -s $SQLSERVER -n $DB --service-objective S0


======================================================
**2 ) destroy_infra.sh**
======================================================
#!/bin/bash

# Set environment variables
REGION="westus"
RGP="day11prem-demo-rg"
CLUSTER_NAME="day11prem-demo-cluster"
ACR_NAME="day11premdemoacr"
SQLSERVER="day11prem-demo-sqlserver"
DB="mhcdb"

# Function to handle errors
handle_error() {
    echo "Error: $1"
    exit 1
}

# Function to check if the resource exists
resource_exists() {
    az resource show --ids $1 &> /dev/null
}

# Delete Azure Kubernetes Service (AKS)
if resource_exists $(az aks show --resource-group $RGP --name $CLUSTER_NAME --query id --output tsv); then
    az aks delete --resource-group $RGP --name $CLUSTER_NAME || handle_error "Failed to delete AKS."
else
    echo "AKS not found. Skipping deletion."
fi

# Delete Azure Container Registry (ACR)
if resource_exists $(az acr show --name $ACR_NAME --resource-group $RGP --query id --output tsv); then
    az acr delete --name $ACR_NAME --resource-group $RGP || handle_error "Failed to delete ACR."
else
    echo "ACR not found. Skipping deletion."
fi

# Delete SQL Database
if resource_exists $(az sql db show --resource-group $RGP --server $SQLSERVER --name $DB --query id --output tsv); then
    az sql db delete --resource-group $RGP --server $SQLSERVER --name $DB || handle_error "Failed to delete SQL Database."
else
    echo "SQL Database not found. Skipping deletion."
fi

# Delete SQL Server
if resource_exists $(az sql server show --resource-group $RGP --name $SQLSERVER --query id --output tsv); then
    az sql server delete --resource-group $RGP --name $SQLSERVER || handle_error "Failed to delete SQL Server."
else
    echo "SQL Server not found. Skipping deletion."
fi

# Delete Resource Group
if resource_exists $(az group show --name $RGP --query id --output tsv); then
    az group delete --name $RGP || handle_error "Failed to delete Resource Group."
else
    echo "Resource Group not found. Skipping deletion."
fi

echo "Resources successfully deleted."
