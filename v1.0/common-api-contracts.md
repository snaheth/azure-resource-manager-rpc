# Common API Contracts
This document describes preferred schemas for resource APIs that are to remain consistent between resource types. 

## Table of Contents ##
- [System Metadata for all Azure resources](#system-metadata-for-all-azure-resources ) </br>
- [Describing location of off-Azure resources](#describing-location-for-off-azure-resources ) </br>
- [Customer-managed Key encryption](#customer-managed-key-encryption)

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

### Resource move ###
Resource move is treated as a combination of creation and deletion. ARM passes one single x-ms-arm-resource-system-data header for the complete move operation. For example, if you have 20 resources to be moved in one request, the expectation from RPs is to save the same system data for each of these 20 resources. In this example,  moving 20 resources is equal to creating 20 resources, and therefore, the systemData values are the same.   

### GDPR Variant Catgeory: Incident Response & Investigations ###
For GDPR, systemData is classified as Incident Response & Investigations data. The data should be tagged with this variant tag, and it will be ignored if someone asks to be forgotten. There are no retention limits; however, the variant tag should be applied to the data as it is being stored.

Data in this asset should only contain End User Identifiable Information, End  User Pseudonymous Identifiers, Customer Account Data, Public Personal data with sufficient privacy tags AND can contain Customer Content Information AND no other GDPR/personal data. Assets can be Azure tenant resource names, End-User IP Addresses, Azure Subscription IDs, and the resource metadata, including any internal Microsoft domain names/machine names or IDs.

| Eligible DSR Variant | Asset Tag in Data Studio |
| ------------- | ------------- |
| - Delete | CandE.NoDelete.SecurityIncidentResponseInvestigation | 
| - Export | CandE.NoExport.SecurityIncidentResponseInvestigation |

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
`locationData` is an optional object property of string fields. Where supported, it should be updatable in both PUT and PATCH methods, and returned on all responses. 

### Properties ###
| Name  | Description |
| ------------- | ------------- |
| name  | a display name for the location  |
| city  | a name of the location city   |
| district | the name of a state, province, or other district name   |
| countryOrRegion | a country or region name for the location. In client experiences, this should not be shortened to "Country." "Country/Region" or "Country or Region"  are appropriate labels. Input values are likely to take one of [these forms](https://en.wikipedia.org/wiki/ISO_3166-1). |


## Customer-managed Key encryption ##
For resources that implement data encryption and allow the customer to specify the key, the preferred API schema is described below. 

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
   "encryption": {
      "status": "enabled",
      "keyVaultProperties": {
        "keyIdentifier": "string",
        "identityClientId": "string"
      }
    },
    "comment": "Resource defined structure"
  }
}
```

### Properties ###
| Name  | Description |
| ------------- | ------------- |
| status  | Enable or disable encryption. Do not include if encryption is mandatory. |
| keyVaultProperties.keyIdentifier  | Key vault uri to access the encryption key  |
| keyVaultProperties.identityClientId | The client id of the identity which will be used to access key vault  |

On PUT/PATCH of a new key, the provider is expected to implement key rotation for the encrypted data. 
