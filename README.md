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

## Creating the Azure Key Vault
```powershell
az keyvault create -n $appname -g $appname
```

## Installing emissary-ingress
```powershell
helm repo add datawire https://app.getambassador.io
helm repo update

kubectl apply -f https://app.getambassador.io/yaml/emissary/3.2.0/emissary-crds.yaml
kubectl wait --timeout=90s --for=condition=available deployment emissary-apiext -n emissary-system

$namespace="emissary"
helm install emissary-ingress datawire/emissary-ingress --set service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"=$appname -n $namespace --create-namespace
kubectl rollout status deployment/emissary-ingress -n $namespace -w

```

## Configuring emissary-ingress routing
```powershell
kubectl apply -f emissary-ingress\listener.yaml -n $namespace
kubectl apply -f emissary-ingress\mappings.yaml -n $namespace 
```

## Installing cert-manager
```powershell
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager --version v1.10.0 --set installCRDs=true --namespace $namespace
```

## Creating the cluster issuer
```powershell
kubectl apply -f cert-manager\cluster-issuer.yaml -n $namespace
kubectl apply -f cert-manager\acme-challenge.yaml -n $namespace
```

## Creating the TLS certificate
```powershell
kubectl apply -f emissary-ingress\tls-certificate.yaml -n $namespace
```

## Enabling tls and https
```powershell
kubectl apply -f emissary-ingress\host.yaml -n $namespace
```

## Packaging and publishing the microservice helm chart
```powershell
helm package helm\microservices

$helmUser=[guid]::Empty.Guid
$helmPassword=az acr login --name $appname --expose-token --output tsv --query accessToken

$env:HELM_EXPERIMENTAL_OCI=1
helm registry login "$appname.azurecr.io" --username $helmUser --password $helmPassword

helm push microservice-0.1.4.tgz oci://$appname.azurecr.io/helm
```

## create github service principal
```powershell
$appId = az ad sp create-for-rbac -n "GitHub" --skip-assignment --query appId --output tsv

az role assignment create --assignee $appId --role "AcrPush" --resource-group $appname
az role assignment create --assignee $appId --role "Azure Kubernetes Service Cluster User Role" --resource-group $appname
az role assignment create --assignee $appId --role "Azure Kubernetes Service Contributor Role" --resource-group $appname
```

## Deploying SEQ to AKS
```powershell
helm repo add datalust https://helm.datalust.co
helm repo update

helm install seq datalust/seq -n observability --create-namespace
```

## Deploying JAEGER to AKS
```powershell
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

helm upgrade jaeger jaegertracing/jaeger --values jaeger\values.yaml -n observability --create-namespace --install
```