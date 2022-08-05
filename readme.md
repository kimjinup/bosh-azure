export USER_NAME=<user_name>
export LOCATION=<region_name>
export VNET_CIDR=<vnet_cidr>
export VNET_NAME=<vnet_name>
export PUBLIC_SUBNET_NAME=<public_subnet_name>
export PUBLIC_SUBNET_CIDR=<public_subnet_cidr>
export RESOURCE_GROUP=<resource_group>
export TENANT_ID=$(az account list --output tsv --query "[].id")
export SUBSCRIPTION_ID=$(az account list --output tsv --query "[?name == '$USER_NAME'].id")
export APP_NAME=<app_name>
export NSG_BOSH_NAME=<bosh_nsg>
export NSG_CF_NAME=<cf_nsg>
export PUBLIC_IP_NAME=<public_ip_name>
export SKU=Basic
export STORAGE_ACCOUNT_NAME=<storage_account_name>



az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.Compute

az group create --name $RESOURCE_GROUP --location $LOCATION

#export APP_ID=$(az ad app create --display-name $APP_NAME --output tsv --query "appId")
#az ad sp create --id $APP_ID
#az role assignment create --role "Owner" --assignee $APP_ID --scope /subscriptions/$SUBSCRIPTION_ID
az ad sp create-for-rbac -n $APP_NAME > ./app_credential.txt



az group create --name $RESOURCE_GROUP --location $LOCATION
az group show --name $RESOURCE_GROUP

az network vnet create --name $VNET_NAME --address-prefixes $VNET_CIDR --resource-group $RESOURCE_GROUP --location $LOCATION
az network vnet subnet create --name $PUBLIC_SUBNET_NAME --address-prefix $PUBLIC_SUBNET_CIDR --vnet-name $VNET_NAME --resource-group $RESOURCE_GROUP
az network vnet show --name boshnet --resource-group $RESOURCE_GROUP

az network nsg create --resource-group $RESOURCE_GROUP --location $LOCATION --name $NSG_BOSH_NAME
az network nsg create --resource-group $RESOURCE_GROUP --location $LOCATION --name $NSG_CF_NAME

az network nsg rule create --resource-group $RESOURCE_GROUP --nsg-name $NSG_BOSH_NAME --access Allow \
   --protocol Tcp --direction Inbound --priority 200 --source-address-prefix Internet --source-port-range '*' \
   --destination-address-prefix '*' --name 'ssh' --destination-port-range 22

az network nsg rule create --resource-group $RESOURCE_GROUP --nsg-name $NSG_BOSH_NAME --access Allow --protocol Tcp \
   --direction Inbound --priority 201 --source-address-prefix Internet --source-port-range '*' \
   --destination-address-prefix '*' --name 'bosh-agent' --destination-port-range 6868

az network nsg rule create --resource-group $RESOURCE_GROUP --nsg-name $NSG_BOSH_NAME --access Allow \
   --protocol Tcp --direction Inbound --priority 202 --source-address-prefix Internet --source-port-range '*' \
   --destination-address-prefix '*' --name 'bosh-director' --destination-port-range 25555

az network nsg rule create --resource-group $RESOURCE_GROUP --nsg-name $NSG_BOSH_NAME --access Allow --protocol '*' \
   --direction Inbound --priority 203 --source-address-prefix Internet --source-port-range '*' \
   --destination-address-prefix '*' --name 'dns' --destination-port-range 53

az network nsg rule create --resource-group $RESOURCE_GROUP --nsg-name $NSG_CF_NAME --access Allow --protocol Tcp \
   --direction Inbound --priority 201 --source-address-prefix Internet --source-port-range '*' \
   --destination-address-prefix '*' --name 'cf-https' --destination-port-range 443

az network nsg rule create --resource-group $RESOURCE_GROUP --nsg-name $NSG_CF_NAME --access Allow --protocol Tcp \
   --direction Inbound --priority 202 --source-address-prefix Internet --source-port-range '*' \
   --destination-address-prefix '*' --name 'cf-log' --destination-port-range 4443

az network public-ip create --name my-public-ip --allocation-method Static --resource-group $RESOURCE_GROUP \
   --location $LOCATION --sku $SKU

az storage account create --name $STORAGE_ACCOUNT_NAME --resource-group $RESOURCE_GROUP --location $LOCATION
export CONNECTION_STRING=$(az storage account show-connection-string \
--name $STORAGE_ACCOUNT_NAME --resource-group $RESOURCE_GROUP \
--output json --query "connectionString")

az storage container create --name bosh --account-name $STORAGE_ACCOUNT_NAME --connection-string $CONNECTION_STRING
az storage container create --name stemcell --account-name $STORAGE_ACCOUNT_NAME --connection-string $CONNECTION_STRING

export ACCOUNT_KEY=$(az storage account keys list --account-name $STORAGE_ACCOUNT_NAME --resource-group $RESOURCE_GROUP --query "[?keyName=='key1'].value" --output tsv)
az storage table create --name stemcells --account-name $STORAGE_ACCOUNT_NAME --account-key $ACCOUNT_KEY

