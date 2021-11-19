# Common API Contracts
This document describes preferred schemas for resource APIs that are to remain consistent between resource types. 

## Table of Contents ##
- [System Metadata for all Azure resources](#system-metadata-for-all-azure-resources) </br>
- [Describing Location for off-Azure resources](#describing-location-for-off-azure-resources)
- [Customer-managed Key encryption](#customer-managed-key-encryption)
- [Singleton Resources](#singleton-resources)

## System Metadata for all Azure resources ##
As of 2020, all Azure resources should implement a read-only `systemData` object property in the top-level envelope. 

```
{
  "id": "/subscriptions/{id}/resourceGroups/{group}/providers/{rpns}/{type}/{name}",
  "name": "{name}",
  "type": "{resourceProviderNamespace}/{resourceType}",
  "location": "North US",
  "systemData":{
      "createdBy": "<string>",
      "createdByType": "<User|Application|ManagedIdentity|Key>",
      "createdAt": "<date-time>",
      "lastModifiedBy": "<string>",
      "lastModifiedByType": "<User|Application|ManagedIdentity|Key>",
      "lastModifiedAt": "<date-time>"
  },
  "tags": {
    "key1": "value 1",
    "key2": "value 2"
  },
  "kind": "resource kind",
  "properties": {
    "comment": "Resource defined structure"
  }
}
```
### Properties ###
| Name  | Description |
| ------------- | ------------- |
| createdBy | a string identifier for the identity that created the resource  |
| createdByType | the type of identity that created the resource: user, application, managedIdentity|
| createdAt | the timestamp of resource creation (UTC) |
| lastModifiedBy | a string identifier for the identity that last modified the resource |
| lastModifiedByType | the type of identity that last modified the resource: user, application, managedIdentity|
| lastModifiedAt | the timestamp of resource last modification (UTC) |


ARM will provide the values in the `x-ms-arm-resource-system-data` header to the provider on resource write and resource action calls in JSON format (as shown above). For ProxyOnly resources, the resource provider is the source of truth. ARM will provide best effort values in the header, but the provider should update the values, including missing values, from known metadata. The `systemData` object should be defined in the resource swagger and persisted in provider storage to serve on all responses for the resource (GET, PUT, PATCH).
**You should only update the systemData properties of a resource if there is a change that affects a user-modifiable property in the control plane JSON configuration of that resource.**

### POST operations ###
For POST operations on tracked resources, ARM will send all the header values. For POST (create) operations on proxy resources, ARM will send all the header values. For POST (action) operations on proxy resources, ARM sends the lastModified* values only. 

### Resource move ###
Resource move is treated as a combination of creation and deletion. The RP should be consistent with this logic and replace all values accordingly.
ARM passes one single x-ms-arm-resource-system-data header for the complete move operation. For example, if you have 20 resources to be moved in one request, the expectation from RPs is to save the same system data for each of these 20 resources. In this example,  moving 20 resources is equal to creating 20 resources, and therefore, the systemData values are the same.   


### FAQ ###
#### Is it OK to collect this data in Microsoft telemetry streams (e.g., Geneva, Kusto, COSMOS, etc.)? ####
**No.** This data is considered Customer Data [Customer Content and End User Identifiable Information (EUII)] and must **NOT** be collected in any telemetry streams in order to meet our data handling requirements. It should only be included in the activity log for and accessible by each individual customer.

#### What values should I use for long-running operations? ####
The systemData properties should reflect the values from when the operation was triggered.

#### Should I replace the values if the operation fails or is cancelled? ####
Do not update system data on a request that was rejected synchronously, but keep the values the triggered-time values if an async operation fails or is cancelled. 
#### What is expected for resources that already exist prior to an RP's implementation of systemData? ####
ARM doesn’t maintain the systemData details and will not fill them up before returning to the client. As a result, as the RP, you are expected to save the values from the first PUT. If the resource already existed prior to your systemData implementation, you can supply null or omit the value when it is not returned by ARM.

#### Should a parent resource's systemData values be impacted by the creation or modification of children resources? ####
No. You should only update the systemData properties of a resource if there is a change that affects a user-modifiable property in the control plane JSON configuration of that resource. Children resources should have their own systemData values.

#### What is the format of the timestamps being sent? What if the value is null (i.e. can’t supply the createdAt)? ####
RPs should either return null or emit the property. 

#### Should internal adminstration operations like updating VM images results in an update of the related resources' systemData properties?
Operations should only change the systemData properties if the change affects user-modifiable properties within the resource's JSON configuration. If a user-modifiable property changes, it should update the systemData values.

#### Can I use a new *ByType value? ####
RPs should only use *ByType values supported by the related enum in the [common types definition](azure-rest-api-specs/types.json at master · Azure/azure-rest-api-specs (github.com)). The currently supported *ByType values are User, Application, ManagedIdentity, and Key.

#### Can I add systemData to my current API version? ####
No. Adding systemData is a breaking change and should only be included as part of a new API version.

#### Is there any situation where the RP will have to interpret the values we are passed and do something? Or do we just store the data and return it? ####
For tracked resources, the values can simply be stored. Since ARM is not aware of all changes to proxy or dataplane-modifiable resources, the values may have to be interpreted for these resource types.

#### How should we store the *ByType values so that we do not have to update the API when more values are added to the identity types? ####
You should store the *ByType values as a string.


## Describing location for off-Azure resources ##
In some cases, resources described in ARM are hosted outside of an Azure datacenter. In this case, an optional `locationData` property is appropriate to allow the user to supply metadata pertaining to the resource geographic location. The locationData property should be in the `properties` object of the resoure.

```
{
  "id": "/subscriptions/{id}/resourceGroups/{group}/providers/{rpns}/{type}/{name}",
  "name": "{name}",
  "type": "{resourceProviderNamespace}/{resourceType}",
  "location": "North US",
   "systemData":{
      "createdBy": "<string>",
      "createdByType": "<User|Application|ManagedIdentity|Key>",
      "createdAt": "<date-time>",
      "lastModifiedBy": "<string>",
      "lastModifiedByType": "<User|Application|ManagedIdentity|Key>",
      "lastModifiedAt": "<date-time>"
  },
  "tags": {
    "key1": "value 1",
    "key2": "value 2"
  },
  "kind": "resource kind",
  "properties": {
    "locationData":{
        "name":"NC-DC-01",
        "city":"Charlotte",
        "district":"NC",
        "countryOrRegion":"USA"
   },
    "comment": "Resource defined structure"
  }
}
```

## Customer-managed Key encryption ##
For resources that implement data encryption and allow the customer to specify the key, the preferred API schema is described below. 

```
{
    "swagger": "2.0",
    "encryption": {
        "type": "object",
        "description": "All encryption configuration for a resource.",
        "properties": {
            "infrastructureEncryption": {
                "type": string,
                "enum": [
                  "enabled",
                  "disabled"
                ],
                "description": "(Optional) Discouraged to include in resource definition. Only needed where it is possible to disable platform (AKA infrastructure) encryption. Azure SQL TDE is an example of this. Values are enabled and disabled.”
            }
            "customerManagedKeyEncryption": {
                "type": "object",
                "description":"All Customer-managed key encryption properties for the resource.",
                "properties": {
                    "keyEncryptionKeyIdentity": {
                        "type": "object",
                        "description":"All identity configuration for Customer-managed key settings defining which identity should be used to auth to Key Vault.",
                        "properties": {
                            "identityType": {
                                "type": string,
                                "enum": [
                                  "systemAssignedIdentity",
                                  "userAssignedIdentity"
                                ],
                                "description": "Values can be systemAssignedIdentity or userAssignedIdentity”
                            },
                            "userAssignedIdentityResourceId": {
                                "type": "string",
                                "defaultValue": "",
                                "description": "user assigned identity to use for accessing key encryption key Url. Ex: /subscriptions/fa5fc227-a624-475e-b696-cdd604c735bc/resourceGroups/<resource group>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myId. Mutually exclusive with identityType systemAssignedIdentity.”
                            },
                            "federatedClientId": {
                                "type": "string",
                                "defaultValue": "",
                                "description": "application client identity to use for accessing key encryption key Url in a different tenant. Ex: f83c6b1b-4d34-47e4-bb34-9d83df58b540”
                            }
                        }
                    },
                    "keyEncryptionKeyUrl": {
                        "type": "string",
                        "defaultValue": "",
                        "description": "key encryption key Url, versioned or unversioned. Ex: https://contosovault.vault.azure.net/keys/contosokek/562a4bb76b524a1493a6afe8e536ee78 or https://contosovault.vault.azure.net/keys/contosokek."
                    }
                }
            }
        }
    }
}
```

### Properties ###
| Name  | Description |
| ------------- | ------------- |
| infrastructureEncryption  | String enum with values of "enabled" and "disabled". It is preferred to make infrastructure or platform encryption mandatory, but if included this enables or disables infrastructure or platform encryption. |
| customerManagedKeyEncryption.keyEncryptionKeyUrl  | Key vault uri to access the encryption key  |
| customerManagedKeyEncryption.keyEncryptionKeyIdentity.identityType | String enum. Values are "userAssignedIdentity" or "systemAssignedIdentity"  |
| customerManagedKeyEncryption.keyEncryptionKeyIdentity.userAssignedIdentity | The User Assigned resource id of the identity which will be used to access key vault  |
| customerManagedKeyEncryption.keyEncryptionKeyIdentity.federatedClientId | The federated application client id of the identity which will be used to access a key vault in another tenant  |

On PUT/PATCH of a new key, the provider is expected to implement key rotation for the encrypted data. 

Error cases:
-	identityType == "systemAssignedIdentity" and userAssignedIdentity != NULL
-	infrastructureEncryption == enabled but keyEncryptionKeyUrl is set. Infrastructure encryption is needed to support CMK.

### State Change Table ###
The following sample ARM requests depict the expected state changes and service behavior.

#### Full Object with System-Assigned Identity ####
```
{
    "encryption": {
        "customerManagedKeyEncryption": {
            "keyEncryptionKeyIdentity": {
                "identityType": "systemAssignedIdentity"
            },
            "keyEncryptionKeyUrl": "https://contosovault.vault.azure.net/keys/contosokek"
        }
    }
} 
```

#### Full Object with User-Assigned Identity ####
```
{
    "encryption": {
        "customerManagedKeyEncryption": {
            "keyEncryptionKeyIdentity": {
                "identityType": "userAssignedIdentity",
                "userAssignedIdentity": "UA resource id"
            },
            "keyEncryptionKeyUrl": "https://contosovault.vault.azure.net/keys/contosokek"
        }
    }
} 
```

#### Full Object with Federated Identity ####
```
{
    "encryption": {
        "customerManagedKeyEncryption": {
            "keyEncryptionKeyIdentity": {
                "identityType": "userAssignedIdentity",
                "userAssignedIdentity": "UA resource id",
                "federatedClientId": "Federated application id"
            },
            "keyEncryptionKeyUrl": "https://contosovault.vault.azure.net/keys/contosokek"
        }
    }
} 
```

#### Partial Patch – System-Assigned to User-Assigned Managed Identity ####
```
{
    "encryption": {
        "customerManagedKeyEncryption": {
            "keyEncryptionKeyIdentity": {
                "identityType": "userAssignedIdentity",
                "userAssignedIdentity": "UA resource id"
            }
        }
    }
} 
```

#### Partial Patch – User-Assigned to System-Assigned Managed Identity ####
```
{
    "encryption": {
        "customerManagedKeyEncryption": {
            "keyEncryptionKeyIdentity": {
                "identityType": "systemAssignedIdentity",
                "userAssignedIdentity": null
            }
        }
    }
}
```

## Singleton Resources ##

There are instances where a resource provider needs to enforce that only a single instance of a proxy resource exists. Many times these resources are configuration related (i.e. [`Microsoft.Sql/servers/databases/securityAlertPolicies`](https://docs.microsoft.com/rest/api/sql/2021-02-01-preview/database-security-alert-policies/create-or-update). Singleton resources should adhere to the following conventions.

### Naming ###

Singleton resources typically have a pre-defined name of `default` or `current`.

### GET APIs ###

Singleton resources must still expose a collection GET and individual GET on the resource type. The collection GET will only return one instance of the resource with the pre-defined resource name.

#### Collection GET Example ####
**URI**: `GET /subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Sql/servers/{serverName}/databases/{databaseName}/securityAlertPolicies`

**Response**: 
```
{
  "value": [
    {
      "id": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Sql/servers/{serverName}/databases/{databaseName}/securityAlertPolicies/default",
      "type": "Microsoft.Sql/servers/databases/securityAlertPolicies",
      "name": "default",
      "properties": {
      }
    }
  ]
}
```

#### Individual GET Example ####
**URI**: `GET /subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Sql/servers/{serverName}/databases/{databaseName}/securityAlertPolicies/default`

**Response**: 
```
{
  "id": "/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Sql/servers/{serverName}/databases/{databaseName}/securityAlertPolicies/default",
  "type": "Microsoft.Sql/servers/databases/securityAlertPolicies",
  "name": "default",
  "properties": {
  }
}
```
