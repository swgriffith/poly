# poly

## Create an Azure Function

```bash
# Create a new project directory
mkdir myfunction
cd myfunction

# Initialize the function project
func init --worker-runtime dotnet

# Create an HTTPTrigger function
func new --template HttpTrigger --name myhttpfunction --authlevel anonymous

# Start the function
func start
```

## Deploy the Azure Function

### To Azure Functions

First we need to create the function app, which includes a resource group, a storage account and the function app.

```bash
# Set ENV Vars
# Function app and storage account names must be unique.
resourceGroup=myRG
storageName=mystorageaccount$RANDOM
functionAppName=myserverlessfunc$RANDOM
region=eastus

# Create a resource group.
az group create --name $resourceGroup --location $region

# Create an Azure storage account in the resource group.
az storage account create \
  --name $storageName \
  --location $region \
  --resource-group $resourceGroup \
  --sku Standard_LRS

# Create a serverless function app in the resource group.
az functionapp create \
  --name $functionAppName \
  --storage-account $storageName \
  --consumption-plan-location $region \
  --resource-group $resourceGroup \
  --functions-version 3
```

Now we can deploy our function

```bash
func azure functionapp publish $functionAppName
```

### To Azure Container Instance

If we're going to deploy to ACI, we first need a container image pushed to a registery where ACI can get it.

```bash
# Have the Functions SDK generate a Dockerfile for us
func init --docker-only

# Set ENV Vars
resourceGroup=myRG
region=eastus
acrName=myacr$RANDOM
# Create a resource group.
az group create --name $resourceGroup --location $region

# Create the Azure Container Registry
az acr create -g $resourceGroup -n $acrName --sku Standard --admin-enabled true
acrFQDN=$(az acr show -g $resourceGroup -n $acrName -o tsv --query loginServer)
acrUserID=$(az acr credential show -g $resourceGroup -n $acrName -o tsv --query username)
acrPasswd=$(az acr credential show -g $resourceGroup -n $acrName -o tsv --query 'passwords[0].value')

# Build the function image
 az acr build -r $acrName -t sample/myapp .
```

Now lets create our container instance!

```bash
# Set ENV Vars
resourceGroup=myRG
region=eastus
# Create a resource group.
az group create --name $resourceGroup --location $region

# Create the instance and grab the FQDN
funcFQDN=$(az container create -g $resourceGroup -n demoapp \
--image $acrFQDN/sample/myapp \
--registry-login-server $acrFQDN \
--registry-username $acrUserID \
--registry-password $acrPasswd \
--ports 80 \
--dns-name-label demoapp$RANDOM \
--query ipAddress.fqdn -o tsv)

# Lets test it!
curl $funcFQDN/api/myhttpfunction\?name=Steve
```

### App Service

```bash
# Set ENV Vars
resourceGroup=myRG
region=eastus
webAppName=demoapp$RANDOM

# Create the app service hosting plan
az appservice plan create --name myAppServicePlan --resource-group $resourceGroup --is-linux

# Create the web app and grab the FQDN
webAppFQDN=$(az webapp create \
--resource-group $resourceGroup \
--plan myAppServicePlan \
--name $webAppName \
--deployment-container-image-name $acrFQDN/sample/myapp:latest \
-o tsv --query 'hostNames[0]')

# Lets test!
curl http://$webAppFQDN/api/myhttpfunction\?name=Steve
```

### Container App


```bash
resourceGroup=myRG
region=canadacentral
logAnalyticsWorkspace="containerapps-logs"
containerAppEnvironment="containerapps-env"

az extension add \
  --source https://workerappscliextension.blob.core.windows.net/azure-cli-extension/containerapp-0.2.0-py2.py3-none-any.whl

az provider register --namespace Microsoft.Web

az group create \
  --name $resourceGroup \
  --location $region

az monitor log-analytics workspace create \
  --resource-group $resourceGroup \
  --workspace-name $logAnalyticsWorkspace

logAnalyticsClientID=`az monitor log-analytics workspace show --query customerId -g $resourceGroup -n $logAnalyticsWorkspace --out tsv`
logAnalyticsSecret=`az monitor log-analytics workspace get-shared-keys --query primarySharedKey -g $resourceGroup -n $logAnalyticsWorkspace --out tsv`

az containerapp env create \
  --name $containerAppEnvironment \
  --resource-group $resourceGroup \
  --logs-workspace-id $logAnalyticsClientID \
  --logs-workspace-key $logAnalyticsSecret \
  --location $region

containerAppFQDN=$(az containerapp create \
  --name my-container-app \
  --resource-group $resourceGroup \
  --environment $containerAppEnvironment \
  --image $acrFQDN/sample/myapp:latest \
  --target-port 80 \
  --ingress 'external' \
  --registry-login-server $acrFQDN \
  --registry-username $acrUserID \
  --registry-password $acrPasswd \
  --query configuration.ingress.fqdn -o tsv)

curl https://$containerAppFQDN/api/myhttpfunction\?name=Steve
```

### AKS Deployment

```bash
# Create an AKS cluster
az aks create -g $resourceGroup -n democluster --generate-ssh-keys 
az aks update -n democluster -g $resourceGroup --attach-acr $(az acr show -g $resourceGroup -n $acrName --query id -o tsv)

az aks get-credentials -g $resourceGroup -n democluster

az acr login -n $acrName 
func kubernetes deploy --name akstest --registry $acrFQDN
```