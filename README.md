# Play.Infra 
Play Economy Infraestructure setup.

## Add the GitHub package source
``` powershell
$owner="DotNetMicroService-Organization"
$ph_pat="[PAT HERE]"

dotnet nuget add source --username USERNAME --password $gh_pat --store-password-in-clear-text --name github "https://nuget.pkg.github.com/$owner/index.json"
```

## Creating the Azure resource group
```powershell
$appname="dotnetplayeconomy"
az group create --name $appname --location eastus
```

## Creating the CosmosDB account
```powershell
az cosmosdb create --name $appname --resource-group $appname --kind MongoDB --enable-free-tier
```

## Creating the Service Bus namespace
```powershell
az servicebus namespace create --name $appname --resource-group $appname --sku Standard
```

## Creating the container Registry
```powershell
az acr create --name $appname --resource-group $appname --sku Basic
```

## Creating the AKS cluster
```powershell
# az feature register --name EnablePodIdentityPreview --namespace Microsoft.ContainerService
# az extension add --name aks-preview

az aks create -n $appname -g $appname --node-vm-size Standard_B2s --node-count 2  --attach-acr $appname --enable-pod-identity --network-plugin azure

az aks get-credentials --name $appname --resource-group $appname
```