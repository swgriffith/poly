# poly

## Initial Setup

We'll start by logging in and creating the resource group we'll be using.

```bash
resourceGroup=containerLabRG
region=eastus

az login
# If running in codespaces
az login --use-device-code

# Find your subscription and set it
az account list
az account set -s <subscriptionid>

az group create -n $resourceGroup -l $region
```

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
storageName=mystorageaccount$RANDOM
functionAppName=myserverlessfunc$RANDOM

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
  --functions-version 4
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
acrName=myacr$RANDOM

# Create the Azure Container Registry
az acr create -g $resourceGroup -n $acrName --sku Premium --admin-enabled true
acrFQDN=$(az acr show -g $resourceGroup -n $acrName -o tsv --query loginServer)
acrUserID=$(az acr credential show -g $resourceGroup -n $acrName -o tsv --query username)
acrPasswd=$(az acr credential show -g $resourceGroup -n $acrName -o tsv --query 'passwords[0].value')

# Build the function image
 az acr build -r $acrName -t sample/myapp .
```

Now lets create our container instance!

```bash
# Create the instance and grab the FQDN
funcFQDN=$(az container create -g $resourceGroup -n demoapp \
--image $acrFQDN/sample/myapp \
--os-type Linux \
--registry-login-server $acrFQDN \
--registry-username $acrUserID \
--registry-password $acrPasswd \
--ip-address Public \
--ports 80 \
--query ipAddress.ip -o tsv)

# Lets test it!
curl $funcFQDN/api/myhttpfunction\?name=Steve
```

### App Service

```bash
# Set ENV Vars
webAppName=demoapp$RANDOM

# Create the app service hosting plan
az appservice plan create --name myAppServicePlan --resource-group $resourceGroup --is-linux

# Create the web app and grab the FQDN
webAppFQDN=$(az webapp create \
--resource-group $resourceGroup \
--plan myAppServicePlan \
--name $webAppName \
--deployment-container-image-name $acrFQDN/sample/myapp:latest \
--docker-registry-server-user $acrUserID \
--docker-registry-server-password $acrPasswd \
-o tsv --query 'hostNames[0]')

# Lets test!
curl http://$webAppFQDN/api/myhttpfunction\?name=Steve
```

### Container App


```bash
logAnalyticsWorkspace="containerapps-logs"
containerAppEnvironment="containerapps-env"

az extension add \
  --source https://workerappscliextension.blob.core.windows.net/azure-cli-extension/containerapp-0.2.0-py2.py3-none-any.whl

az provider register --namespace Microsoft.Web

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
  --location canadacentral

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

az aks get-credentials -g $resourceGroup -n democluster

acrFQDN=$(az acr show -g $resourceGroup -n $acrName -o tsv --query loginServer)
acrUserID=$(az acr credential show -g $resourceGroup -n $acrName -o tsv --query username)
acrPasswd=$(az acr credential show -g $resourceGroup -n $acrName -o tsv --query 'passwords[0].value')

kubectl create secret docker-registry regcred \
--docker-server=$acrFQDN \
--docker-username=$acrUserID \
--docker-password=$acrPasswd

az acr login -n $acrName 

# Lets have a look at the deployment manifest
func kubernetes deploy --name akstest --pull-secret regcred --registry $acrFQDN --dry-run > deployment.yaml

# Now lets run it
func kubernetes deploy --name akstest --pull-secret regcred --registry $acrFQDN
```