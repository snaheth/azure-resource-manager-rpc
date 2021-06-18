# Addendum
## Table of Contents ##

- [Instrumentation and Tracing across services](#instrumentation-and-tracing-across-services) <br/>
- [ETags for Resources](#etags-for-resources) <br/>
- [Regional Endpoints](#regional-endpoints) <br/>
- [Nested Resources](#nested-resources) <br/>
- [Resource Group Deletes](#resource-group-deletes) <br/>
- [Retrying REST Calls](#retrying-rest-calls)
- [Designing Resources](#designing-resources)
- [Enumerating SKUs for an existing resource](#enumerating-skus-for-an-existing-resource)
- [Enumerating SKUs for a new resource](#enumerating-skus-for-a-new-resource)
- [Correlating resources created on behalf of customer](#correlating-resources-created-on-behalf-of-customer)


## Instrumentation and Tracing across services ## 

Resource providers should associate the correlation Id and client request Id with all of their management operations.

This will enable end-to-end troubleshooting from the portal, through CSM and to the RP, via four key headers:

1. x-ms-client-request-id: associated with a logical action in the client; e.g. loading the portal monitoring part may have a single client request id for many calls to ARM (e.g. load VMs + load metric definitions + load metrics)
2. x-ms-correlation-request-id: associated with a logical action in ARM; e.g. deleting a resource group or running a template
3. x-ms-request-id: returned by the RP per request; typically maps to the RP's ActivityId
4. x-ms-routing-request-id: maps to ARM's activity id (which is not otherwise exposed). This maps to implementation boundaries in ARM / the platform –RPs should not use this as their ActivityId.

## ETags for Resources ##

ETags may be returned for individual resources, and then sent via If-Match / If-None-Match headers for concurrency control. The resource provider is responsible for storing and validating ETags – the frontdoor will never inspect these values.

The behavior for ETags can be seen below:

| PUT | Resource does not exist | Resource exists |
| --- | --- | --- |
| If-Match = "" / absent | 201 Created | 200 OK |
| If-Match = "\*" | 412* Precondition Failed | 200 OK |
| If-Match = "xyz" | 412** Precondition Failed | 200 OK / 412 Precondition Failed |
| If-None-Match = "\*" | 201 Created | 412**** Precondition Failed |


| PATCH | Resource does not exist | Resource exists |
| --- | --- | --- |
| If-Match = "" / absent | 404 Not Found | 200 OK |
| If-Match = "\*" | 404** Not Found | 200 OK |
| If-Match = "xyz" | 404** Not Found | 200 OK / 412 Precondition Failed |

| DELETE | Resource does not exist | Resource exists |
| --- | --- | --- |
| If-Match = "" / absent | 204 No Content | 200 OK |
| If-Match = "\*" | 204*** No Content | 200 OK |
| If-Match = "xyz" | 204*** No Content | 200 OK / 412 Precondition Failed |

## Regional Endpoints ##

The Resource Provider should partition services across multiple regions in order to allow for continued service if there is a regional outage. It is recommended that the RP offer services in all Azure supported regions so that users may collocate interdependent resources within a resource group.

The RP should register all the regions that they support, and should provide a different regional endpoint for each region. This regional endpoint and its backing state/data should be isolated from the other regions in order to isolate failure within Azure. ARM will proxy the request to the appropriate region based on the resource&#39;s location.

The resource provider should register regional DNS names for each regional endpoint. The naming of this region should follow the below convention:

1. {region}.{provider namespace}.azure.com
2. {region}.{subservice name}.{provider namespace}.azure.com

In a concrete form:

1. westus.cache.api.azure.com, eastus.cache.api.azure.com
2. westus.redis.cache.api.azure.com

## Nested Resources ##

The RPC supports registering "nested" resource types for an RP with this format:

 _{RPNS}/{TYPE1}/{NAME1}/{TYPE2}/{NAME2}._

The resource provider URIs are still expected to follow REST guidelines (e.g.: azure.com/subscriptions/{SubId}/resourceGroup/ {resourceGroup\_name}/providers/{namespace}/{type1}/{type1\_name}/{type2}/{type2\_name}

Please see example URLs for nested resource types below:

| Http Method | URI |
| --- | --- |
| PUT | https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{namespace}/{resourceType}/{resourceName}/{nestedResourceType}/{nestedResourceName}?api-version={api-version} |
| GET | https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{namespace}/{resourceType}/{resourceName}/{nestedResourceType}/{nestedResourceName}?api-version={api-version} |
| GET | https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{namespace}/{resourceType}/{resourceName}/{nestedResourceType}?api-version={api-version} |
| PATCH | https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{namespace}/{resourceType}/{resourceName}/{nestedResourceType}/{nestedResourceName}?api-version={api-version} |
| DELETE | https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{namespace}/{resourceType}/{resourceName}/{nestedResourceType}/{nestedResourceName}?api-version={api-version} |

## Resource Group Deletes ##

If the resource group containing resources is deleted, the RP will receive a DELETE call on each individual tracked resource in that group. In some cases, the resources will have interdependencies and therefore can only be deleted in a certain order that ARM does not understand.

To address this, ARM can simply ask each resource to delete in arbitrary order. If the resource refuses the deletion, then ARM will revisit it after trying to delete the rest of the group&#39;s resources. It will continue to revisit resources until all tracked resources are deleted, or there is no further delete progress (for example if a delete fails because of an ACL).

As a result, it is essential that the RPs follow proper REST guidelines when returning 2xx for a successful delete, and 4xx for a delete that is invalid (until a related resource is deleted first).

## Retrying REST Calls ##

The recommended pattern for REST clients is to retry calls that:

1. Return 408 indicating a request timeout;
2. Return a 5xx status code indicating a server failure (e.g. 500 InternalServerError; 503 Service Unavailable);
3. Experience a socket exception (e.g. DNS resolution failures; network drop; no listening socket).
4. Any other response where a Retry-After header is present.

The retry interval, strategy (e.g. exponential/linear) and max timeout should be defined by the _server_ and not the client. The server indicates how long it expects the client to wait before retrying the call by including a (Retry-After header)[https://tools.ietf.org/html/rfc7231#section-7.1.3] in the response. That allows various server workloads to advertise different retry policies.

## Designing Resources ##

The Azure Resource Provider contract and template language have special semantics around &quot;registered&quot; Azure resources – we call these **tracked resources**.

If these objects are addressable by URIs but not "registered", we call these **proxy only** resources to differentiate them.

**Background REST philosophy**

1. Azure API guideline compliant and OData compliant where it makes sense.
2. APIs must be callable from any client implementation, language or framework.
3. Making things simple and regular for machine processing often makes things easier for humans to understand, even when it increases verbosity.
4. When we choose between understandable URLs and understandable templates, we should make the templates human understandable and the URLs machine consumable.
5. Everything that is possible from a template should be possible through imperative APIs, but not necessarily as easily and may require special client code (orchestration is an obvious example).

**Objects should be Tracked resources if:**

1. They are billable entities, including freemium models
2. Need to be orchestrated by the template execution engine
3. Related to above, they have dependencies on other resources

**Objects should be top-level resources if:**

1. They should be resources as per above
2. Their lifetime is independent of other resources (only dependent on the containing resource group)
3. If the objects have an independent lifetime but are not important to the model, they should only be top-level resources if no other design can be made to work.

**Objects should be nested resources if:**

1. They meet the guidelines for Azure resources above (billable, orchestration supported, taggable)
2. Their lifetime is dependent on a containing resource

**Objects should be proxy only resources if:**

1. They don&#39;t need to be Azure resources for one of the reasons above
2. if the objects/properties are required for the parent resource to be created, then they should \*not\* be Azure resources
3. It is important that they be simple to create in context of a containing resource.
4. The lifecycle of the resources are managed outside ARM (e.g. agent based discovery).
5. They are very common and would incur a performance overhead on the system (for example by cluttering the indices and GET resources calls)

**Comparison of Tracked Resources and proxy only Resources**

| ​ **Feature** | ​ **Tracked Resources** | ​ **Proxy only Resources** |
| --- | --- | --- |
| ​Understood by template processor | ​Yes | ​ Yes |
| Can be top-level resource | Yes | Only if global/single region |
| Defined JSON format to payload | ​Yes -- must comply with general Azure resource format (id, type, name, tags, properties, location etc.) | ​Yes -- must comply with general Azure resource format (id, type, name, properties, etc.) but no tags or location |
| Defined URL format | ​Yes -- must use providers/{namespace}/{typename} URL | ​Yes -- must use providers/{namespace}/{typename} URL |
| ​Tagging and Indexing supported | ​Yes | ​No |
| ​Billing supported     | ​Yes | ​No |
| ​Have their own URLs | ​Yes | ​Yes |
| ​​Can be PUT (created) with the parent in a single call | ​No - $batch/deep insert not available | ​​Yes, see deep insert recommendation |
| ​Supports RBAC | ​Yes -- based on URL     | ​ Yes -- based on URL |
| ​Actions are advertised and supported by RBAC experience (/reboot for example) | ​Yes | ​Optional |
| ​Supports targeted imperative GET | ​Yes | ​ Yes |
| ​Supports targeted imperative PUT, PATCH | ​Yes | ​ Yes |
| ​Supports targeted DELETE | ​Yes | ​Yes |
| ​DELETEd when parent is | ​Yes, governed by deletionPolicy in Manfiest | ​RP must implement cascading delete and/or validation |
| Independent lifetime from parent | ​DELETEd when parent is, but otherwise independent | ​ RP must implement cascading delete and/or validation, but otherwise independent |
| ​Supported by orchestration service         | ​Yes | ​ Yes |
| ​Can take dependencies (DependsOn, [reference] on other resources     | ​Yes     | ​Yes |
| ​Can be the target of dependencies from other resources | ​Yes | ​Yes |
| ​In template can take values from variables | ​Yes | ​Yes |
| ​In template can take values from other resources | ​Yes | ​Yes |
| ​Can be independently included in external templates | ​Yes | ​Yes |
| ​Can be versioned independently from parent     | ​Yes | ​Yes |

## Enumerating SKUs for an existing resource ##

In many cases, it is desirable for users to resize an existing resource.  However, only a certain subset of sizes available from a given resource provider may be applicable to a given resource – e.g. an A1 machine may be updatable to other A series VMs, but may not be convertible to D series.  This API exposes the valid SKUs for an existing resource.

### Request ###

| Method | Request URI |
| --- | --- |
| GET | https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{resourceProviderNamespace}/{resourceType}/{resourceName}/skus?api-version={api-version} |

**Arguments**

| Argument | Description |
| --- | --- |
| subscriptionId | The subscriptionId for the Azure user. |
| resourceGroupName | The resource group name uniquely identifies the resource group within the user subscriptionId. |
| resourceType | The type of the resource – the resource providers declare the resource types they support at the time of registering with Azure. |
| resourceName | The name of the resource. |
| api-version | Specifies the version of the protocol used to make this request. Format must match YYYY-MM-DD[-preview|-alpha|-beta|-rc|-privatepreview]. |

**Request Headers**

Only headers common to all requests.

**Request Body**

Empty

### Response ###

The response includes an HTTP status code, a set of response headers, and a response body.

**Status Code**

The resource provider should return 200 (OK) to indicate that the operation completed successfully.

If the resource does not exist, 404 (NotFound) should be returned. For other errors (e.g. internal errors) use the appropriate HTTP error code.  If no skus are valid, return 200 + an empty value array.

**Response Headers**

Headers common to all responses.

**Response Body**

    {
      "value": [
        {
          "resourceType": "type for sku"
          "sku": {
            "name": "sku name",
            "tier": "sku tier"
          },

          "capacity": {
            "minimum": min,
            "maximum": max,
            "default": default,
            "scaleType": "automatic|manual|none"
          }
        },

        {
          "resourceType": "type for sku"
          "sku": {
            "name": "sku name",
            "tier": "sku tier"
          },

          "capacity": {
            "minimum": min,
            "maximum": max,
            "default": default,
            "scaleType": "automatic|manual|none"
          }
        }
      ]
    }

For a detailed explanation of each field in the response body, please refer to the request body description in the PUT resource section. The only GET specific properties are "name", "type" and "id"

| Field | Description |
| --- | --- |
| resourceType | Required, string. The resource type that this object applies to. For example, for a database it'd be: Microsoft.SQL/servers/databases.   |
| sku | Required, object. The exact set of keys that define this sku. This matches the fields on the respective resource. |
| sku.name | Required, string. The name of the SKU. This is typically a letter + number code, such as A0 or P3 |
| sku.tier | Optional, string. The tier of this particular SKU. Typically one of: <ul> <li>Free</li> <li>Basic</li> <li>Standard</li> <li>Premium</li></ul>|
| capacity | Optional, object. If the SKU supports scale out/in then the capacity object must be included. If scale out/in is not possible for the resource this may be omitted. |
| capacity.scaleType | Required, enum. One of: <ul><li>None – meaning the capacity is not adjustable in any way (e.g. it is fixed)</li> <li>Manual – the user must manually scale out/in</li> <li>Automatic – the user is permitted to scale this SKU in and out</li></ul>|
| capacity.minimum | Required if scaleType != none, integerThe lowest permitted capacity for this resource. Typically 0 or 1. |
| capacity.maximum | Required if scaleType != none, integerThe highest permitted capacity for this resource. This may vary per SKU, or it may be the same. |
| capacity.default | Required if scaleType != none, integerThe default capacity.  Generally, if the user omits setting the capacity, the RP would use this value. |

## Enumerating SKUs for a new resource ##

For new resources, SKUs are enumerated via ARM's metadata store.  This allows users to enumerate SKUs based on location, api-version, feature flags, and offerId filters.

## Correlating resources created on behalf of customer ##
It is required that Resource Providers (RP) tag the resources when making service-to-service (S2S) calls using their internal subscription. When a customer call to provision a resource comes to a RP and that RP needs to call another RP using an internal subscription to complete the request, it must tag the resource(s) with the customer’s original fullyqualified resourceId. This tag will not be visible to the customers as it will reside in the internal subscription only. This is applicable for both PUT andPATCH.

The tag should have the following format –
Tag Key – resourceCreatedFor
TagValue – {fullyqualified resourceId of the resourcerequested by the customer. Ex-/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/{resourceProviderNamespace}/{resourceType}/{resourcename}}

Example – When a customer requests an HDInsight cluster with
7 workers, HDI RP will call CRP to provision 7 VMs. It is now required that HDI
RP will add the tags to the request for creating the 7 VMs.

