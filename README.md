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