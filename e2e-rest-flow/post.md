# Azure Data Share configuration using REST APIs

In this quickstart, you will configure Azure Data Share through REST APIs for Provider and Consumer integration flows. You will use Azure Data Lake Store Gen 2 as the source and destination locations and configure Data Share to transfer data to Consumers on an hourly basis.

REST APIs will be invoked using `az rest` command.  This simplies the quickstart by delegating the credential management, request/response flows and error handling.  `az rest` will use the credentials (i.e. OAuth token) from the account that's logged in via `az login` to invoke the requests.  You can learn more from the [Azure CLI documentation](https://docs.microsoft.com/cli/azure/reference-index?view=azure-cli-latest#az-rest).

## Prequisites

For the purposes of this quickstart, you will need to configure 2 Resource Groups, each with an [Azure Data Lake Store Gen 2 (ADLS Gen 2)](https://docs.microsoft.com/azure/storage/blobs/data-lake-storage-quickstart-create-account).  Following Azure CLI commands can be used to create these resources.

### Create Resource Groups

```bash
az group create -n ads-demo-provider -l eastus2

az group create -n ads-demo-consumer -l eastus2
```

### Create ADLS Gen 2 Storage Accounts

```bash
# Add CLI extension for ADLS Gen 2
az extension add --name storage-preview

# Create provider Storage Account
az storage account create --sku Standard_LRS --kind StorageV2 --hierarchical-namespace true -l eastus2 -g ads-demo-provider -n adlsprovider

# Create consumer Storage Account
az storage account create --sku Standard_LRS --kind StorageV2 --hierarchical-namespace true -l eastus2 -g ads-demo-consumer -n adlsconsumer
```

### Setup filesystem and seed data to the provider storage account

```bash
# Retrieve the storage account connection string for provider storage account
export AZURE_STORAGE_CONNECTION_STRING=`az storage account show-connection-string -g ads-demo-provider -n adlsprovider -o json --query "connectionString"`

# Create a filesystem
az storage container create --name datasetfs

# Upload files from local machine - change the source to your location and --destination-path is a subpath under the filesystem (i.e. path will be datasetfs/scripts/terraform)
az storage blob upload-batch --source . --destination datasetfs --destination-path scripts/terraform
```

### Setup filestem for the consumer storage account

```bash
# Retrieve the storage account connection string for provider storage account
export AZURE_STORAGE_CONNECTION_STRING=`az storage account show-connection-string -g ads-demo-consumer -n adlsconsumer -o json --query "connectionString"`

# Create a filesystem
az storage container create --name datasetfs
```

## Configure Azure Data Share for Provider

### Step 1: Set environment variables
```bash
# Set Default Values
export PROVIDER_SUBSCRIPTION_ID=""
export PROVIDER_RESOURCE_GROUP="ads-demo-provider"
export PROVIDER_DATASHARE_ACCOUNT_NAME="adsdemoprovider"
export PROVIDER_DATASHARE_SHARE_NAME="scriptshare"
export PROVIDER_DATASHARE_SHARE_DATASET_NAME="dataset1"
export PROVIDER_ADLSGEN2_NAME="adlsprovider"
export PROVIDER_ADLSGEN2_FS="datasetfs"
export PROVIDER_ADLSGEN2_DATASET_PATH="/scripts"
export CONSUMER_EMAIL_ADDRESS=""
```

### Step 2: Create Azure Data Share account

[Reference](https://docs.microsoft.com/en-us/rest/api/datashare/accounts/create)

JSON Payload

```json
{
  "location": "eastus2",
  "identity": {
    "type": "SystemAssigned"
  }
}
```

REST API call

```bash
az rest -m PUT -u "https://management.azure.com/subscriptions/$PROVIDER_SUBSCRIPTION_ID/resourceGroups/$PROVIDER_RESOURCE_GROUP/providers/Microsoft.DataShare/accounts/$PROVIDER_DATASHARE_ACCOUNT_NAME?api-version=2018-11-01-preview" --body "{\"location\": \"eastus2\", \"identity\": { \"type\": \"SystemAssigned\"}}"
```

### Step 3: Configure read permissions for Azure Data Share account

Please follow the instructions to [setup role assignment](https://docs.microsoft.com/en-us/azure/data-share/concepts-roles-permissions#data-providers) such that Data Share Account can **read** data from the ADLS Gen 2 storage account.

### Step 4: Create Share

[Reference](https://docs.microsoft.com/en-us/rest/api/datashare/shares/create)

JSON Payload

```json
{
  "properties": {
    "description": "Share description",
    "terms": "Confidential",
    "shareKind": "CopyBased"
  }
}
```

REST API Call

```bash
az rest -m PUT -u "https://management.azure.com/subscriptions/$PROVIDER_SUBSCRIPTION_ID/resourceGroups/$PROVIDER_RESOURCE_GROUP/providers/Microsoft.DataShare/accounts/$PROVIDER_DATASHARE_ACCOUNT_NAME/shares/$PROVIDER_DATASHARE_SHARE_NAME?api-version=2018-11-01-preview" --body "{ \"properties\": { \"description\": \"Share description\", \"terms\": \"Confidential\", \"shareKind\": \"CopyBased\" } }"
```

### Step 5: Create Data Set

[Reference](https://docs.microsoft.com/en-us/rest/api/datashare/datasets/create)

JSON Payload

```json
{
  "kind": "AdlsGen2Folder",
  "properties": {
    "storageAccountName": "$PROVIDER_ADLSGEN2_NAME",
    "resourceGroup": "$PROVIDER_RESOURCE_GROUP",
    "fileSystem": "$PROVIDER_ADLSGEN2_FS",
    "folderPath": "$PROVIDER_ADLSGEN2_DATASET_PATH",
    "subscriptionId": "$PROVIDER_SUBSCRIPTION_ID"
  }
}
```

REST API Call

```bash
az rest -m PUT -u "https://management.azure.com/subscriptions/$PROVIDER_SUBSCRIPTION_ID/resourceGroups/$PROVIDER_RESOURCE_GROUP/providers/Microsoft.DataShare/accounts/$PROVIDER_DATASHARE_ACCOUNT_NAME/shares/$PROVIDER_DATASHARE_SHARE_NAME/dataSets/$PROVIDER_DATASHARE_SHARE_DATASET_NAME?api-version=2018-11-01-preview" --body "{ \"kind\": \"AdlsGen2Folder\", \"properties\": { \"storageAccountName\": \"$PROVIDER_ADLSGEN2_NAME\", \"resourceGroup\": \"$PROVIDER_RESOURCE_GROUP\", \"fileSystem\": \"$PROVIDER_ADLSGEN2_FS\", \"folderPath\": \"/$PROVIDER_ADLSGEN2_DATASET_PATH\", \"subscriptionId\": \"$PROVIDER_SUBSCRIPTION_ID\" } }"
```


### Step 6: Create Synchronization Schedule

[Reference](https://docs.microsoft.com/en-us/rest/api/datashare/synchronizationsettings/create)

JSON Payload

```json
{
  "kind": "ScheduleBased",
  "properties": {
    "synchronizationTime": "2019-01-01T01:00:52.9614956Z",
    "recurrenceInterval": "Hour",
    "synchronizationMode": "Incremental"
  }
}
```

REST API Call

```bash
az rest -m PUT -u "https://management.azure.com/subscriptions/$PROVIDER_SUBSCRIPTION_ID/resourceGroups/$PROVIDER_RESOURCE_GROUP/providers/Microsoft.DataShare/accounts/$PROVIDER_DATASHARE_ACCOUNT_NAME/shares/$PROVIDER_DATASHARE_SHARE_NAME/synchronizationSettings/hourlysync?api-version=2018-11-01-preview" --body "{  \"kind\": \"ScheduleBased\",   \"properties\": {     \"synchronizationTime\": \"2019-01-01T01:00:52.9614956Z\",     \"recurrenceInterval\": \"Hour\",     \"synchronizationMode\": \"Incremental\"  }}"
```

### Step 7: Create Invitation

[Reference](https://docs.microsoft.com/en-us/rest/api/datashare/invitations/create)

JSON Payload

```json
{
  "properties": {
    "targetEmail": "$CONSUMER_EMAIL_ADDRESS"
  }
}
```

REST API Call

```bash
az rest -m PUT -u "https://management.azure.com/subscriptions/$PROVIDER_SUBSCRIPTION_ID/resourceGroups/$PROVIDER_RESOURCE_GROUP/providers/Microsoft.DataShare/accounts/$PROVIDER_DATASHARE_ACCOUNT_NAME/shares/$PROVIDER_DATASHARE_SHARE_NAME/invitations/invite-user?api-version=2018-11-01-preview" --body "{ \"properties\": { \"targetEmail\": \"$CONSUMER_EMAIL_ADDRESS\" } }"
```




## Configure Azure Data Share for Consumer

### Step 1: Set environment variables
```bash
# Set Default Values
export CONSUMER_SUBSCRIPTION_ID=""
export CONSUMER_RESOURCE_GROUP="ads-demo-consumer"
export CONSUMER_DATASHARE_ACCOUNT_NAME="adsdemoconsumer"
export CONSUMER_ADLSGEN2_NAME="adlsconsumer"
export CONSUMER_ADLSGEN2_FS="datasetfs"
export CONSUMER_ADLSGEN2_DATASET_PATH="/shared-data/scripts"
```

### Step 2:  Create Azure Data Share for Consumer

[Reference](https://docs.microsoft.com/en-us/rest/api/datashare/accounts/create)

JSON Payload

```json
{
  "location": "eastus2",
  "identity": {
    "type": "SystemAssigned"
  }
}
```

REST API call

```bash
az rest -m PUT -u "https://management.azure.com/subscriptions/$CONSUMER_SUBSCRIPTION_ID/resourceGroups/$CONSUMER_RESOURCE_GROUP/providers/Microsoft.DataShare/accounts/$CONSUMER_DATASHARE_ACCOUNT_NAME?api-version=2018-11-01-preview" --body "{\"location\": \"eastus2\", \"identity\": { \"type\": \"SystemAssigned\"}}"
```

### Step 2: Configure write permissions for Azure Data Share account

Please follow the instructions to [setup role assignment](https://docs.microsoft.com/en-us/azure/data-share/concepts-roles-permissions#data-consumers) such that Data Share Account can **write** data from the ADLS Gen 2 storage account.


### Step 3: Identify invitations

[Reference](https://docs.microsoft.com/en-us/rest/api/datashare/consumerinvitations/listinvitations)

REST API Call

```bash
az rest -m GET -u "https://management.azure.com/providers/Microsoft.DataShare/ListInvitations?api-version=2018-11-01-preview" --output-file /tmp/invitations-output.json

cat /tmp/invitations-output.json
```

Sample Output

```json
{
  "value": [
    {
      "properties": {
        "description": "Share description",
        "dataSetCount": 1,
        "invitationId": "f2b219eb-6378-4237-8ca1-28cc5cc53599",
        "invitationStatus": "Pending",
        "location": "eastus2",
        "sender": "John Doe",
        "senderCompanyName": "Microsoft",
        "shareName": "scriptshare",
        "sentAt": "2019-10-25T17:57:35.3448859Z",
        "termsOfUse": "Confidential"
      },
      "id": "/providers/Microsoft.DataShare/consumerInvitations/f2b219eb-6378-4237-8ca1-28cc5cc53599",
      "name": "invite-user",
      "type": "Microsoft.DataShare/consumerInvitations"
    }
  ]
}
```

### Step 4:  Set additional environment variables based on invitation

```bash
# value from shareName attribute in the response 
export CONSUMER_SOURCE_SHARE_NAME="scriptshare"
export CONSUMER_INVITATION_ID="f2b219eb-6378-4237-8ca1-28cc5cc53599"
```


### Step 5:  Subscribe to the invitation

[Reference](https://docs.microsoft.com/en-us/rest/api/datashare/sharesubscriptions/create)

JSON Payload

```json
{
  "properties": {
    "invitationId": "$CONSUMER_INVITATION_ID"
  }
}
```

REST API Call

```bash
az rest -m PUT -u "https://management.azure.com/subscriptions/$CONSUMER_SUBSCRIPTION_ID/resourceGroups/$CONSUMER_RESOURCE_GROUP/providers/Microsoft.DataShare/accounts/$CONSUMER_DATASHARE_ACCOUNT_NAME/shareSubscriptions/$CONSUMER_SOURCE_SHARE_NAME?api-version=2018-11-01-preview" --body "{ \"properties\": { \"invitationId\": \"$CONSUMER_INVITATION_ID\" } }"
```

### Step 6:  Identify data sets available on the share

[Reference](https://docs.microsoft.com/en-us/rest/api/datashare/consumersourcedatasets/listbysharesubscription)

REST API Call

```bash
az rest -m GET -u "https://management.azure.com/subscriptions/$CONSUMER_SUBSCRIPTION_ID/resourceGroups/$CONSUMER_RESOURCE_GROUP/providers/Microsoft.DataShare/accounts/$CONSUMER_DATASHARE_ACCOUNT_NAME/shareSubscriptions/$CONSUMER_SOURCE_SHARE_NAME/ConsumerSourceDataSets?api-version=2018-11-01-preview" --output-file /tmp/datasets-on-share-output.json

cat /tmp/datasets-on-share-output.json
```

Sample Output

```json
{
  "value": [
    {
      "properties": {
        "dataSetName": "//scripts",
        "dataSetId": "49d3b8b8-1a79-48c2-86cb-2cec1519b817",
        "dataSetType": "AdlsGen2Folder"
      },
      "id": "/subscriptions/$CONSUMER_SUBSCRIPTION_ID/resourceGroups/$CONSUMER_RESOURCE_GROUP/providers/Microsoft.DataShare/accounts/adsdemoconsumer/$CONSUMER_DATASHARE_ACCOUNT_NAME/scriptshare/consumerSourceDataSets/dataset1",
      "name": "dataset1",
      "type": "Microsoft.DataShare/ConsumerSourceDataSet"
    }
  ]
}
```

### Step 7:  Set additional environment variables based on invitation

```bash
# value from properties.datasetId
export CONSUMER_SOURCE_DATASET_ID="49d3b8b8-1a79-48c2-86cb-2cec1519b817"
```

### Step 8:  Create Data Set mapping to destination ADLS Gen 2 Storage ACcount

[Reference](https://docs.microsoft.com/en-us/rest/api/datashare/datasetmappings/create)

JSON Payload

```json
{
  "kind": "AdlsGen2Folder",
  "properties": {
    "datasetId": "$CONSUMER_SOURCE_DATASET_ID",
    "storageAccountName": "$CONSUMER_ADLSGEN2_NAME",
    "folderPath": "$CONSUMER_ADLSGEN2_DATASET_PATH",
    "fileSystem": "$CONSUMER_ADLSGEN2_FS",
    "subscriptionId": "$CONSUMER_SUBSCRIPTION_ID",
    "resourceGroup": "$CONSUMER_RESOURCE_GROUP"
  }
}
```

REST API Call

```bash
az rest -m PUT -u "https://management.azure.com/subscriptions/$CONSUMER_SUBSCRIPTION_ID/resourceGroups/$CONSUMER_RESOURCE_GROUP/providers/Microsoft.DataShare/accounts/$CONSUMER_DATASHARE_ACCOUNT_NAME/shareSubscriptions/$CONSUMER_SOURCE_SHARE_NAME/dataSetMappings/mapping?api-version=2018-11-01-preview" --body "{ \"kind\": \"AdlsGen2Folder\", \"properties\": { \"datasetId\": \"$CONSUMER_SOURCE_DATASET_ID\", \"storageAccountName\": \"$CONSUMER_ADLSGEN2_NAME\",\"folderPath\": \"$CONSUMER_ADLSGEN2_DATASET_PATH\", \"fileSystem\": \"$CONSUMER_ADLSGEN2_FS\", \"subscriptionId\": \"$CONSUMER_SUBSCRIPTION_ID\", \"resourceGroup\": \"$CONSUMER_RESOURCE_GROUP\" }}"
```

### Step 9:  List data synchronization settings

[Reference](https://docs.microsoft.com/en-us/rest/api/datashare/sharesubscriptions/listsourcesharesynchronizationsettings)


REST API Call

```bash
az rest -m post -u "https://management.azure.com/subscriptions/$CONSUMER_SUBSCRIPTION_ID/resourceGroups/$CONSUMER_RESOURCE_GROUP/providers/Microsoft.DataShare/accounts/$CONSUMER_DATASHARE_ACCOUNT_NAME/shareSubscriptions/$CONSUMER_SOURCE_SHARE_NAME/listSourceShareSynchronizationSettings?api-version=2018-11-01-preview" --output-file /tmp/sync-settings-output.json

cat /tmp/sync-settings-output.json
```

Sample Output

```json
{
  "value": [
    {
      "properties": {
        "recurrenceInterval": "Hour",
        "synchronizationTime": "2019-01-01T01:00:52.9614956Z"
      },
      "kind": "ScheduleBased"
    }
  ]
}
```

### Step 10:  Set additional environment variables based on invitation

```bash
# value from properties.datasetId
export CONSUMER_SYNC_RECURRENCE="Hour"
export CONSUMER_SYNC_TIME="2019-01-01T01:00:52.9614956Z"
```

### Step 11:  Trigger incremental synchronization

[Reference](https://docs.microsoft.com/en-us/rest/api/datashare/triggers/create)

JSON Payload

```json
{
  "kind": "ScheduleBased",
  "properties": {
    "synchronizationTime": "$CONSUMER_SYNC_TIME",
    "recurrenceInterval": "$CONSUMER_SYNC_RECURRENCE",
    "synchronizationMode": "Incremental"
  }
}
```

REST API Call

```bash
az rest -m PUT -u "https://management.azure.com/subscriptions/$CONSUMER_SUBSCRIPTION_ID/resourceGroups/$CONSUMER_RESOURCE_GROUP/providers/Microsoft.DataShare/accounts/$CONSUMER_DATASHARE_ACCOUNT_NAME/shareSubscriptions/$CONSUMER_SOURCE_SHARE_NAME/triggers/fullcopytrigger?api-version=2018-11-01-preview" --body "{ \"kind\": \"ScheduleBased\", \"properties\": { \"synchronizationTime\": \"$CONSUMER_SYNC_TIME\", \"recurrenceInterval\": \"$CONSUMER_SYNC_RECURRENCE\", \"synchronizationMode\": \"Incremental\" } }"
```