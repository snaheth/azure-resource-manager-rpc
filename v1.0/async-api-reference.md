# Asynchronous API Reference

- [Asynchronous Operations](#asynchronous-operations) <br/>
- [Creating or Updating Resources Asynchronously](#creating-or-updating-resources-asynchronously)
- [Delete Resource Asynchronously](#delete-resource-asynchronously)
- [Call Action POST Asynchronously](#call-action-post-asynchronously)
- [ProvisioningState property](#provisioningstate-property)
- [202 Accepted and Location Headers](#202-accepted-and-location-headers)
- [Azure-AsyncOperation Resource format](#azure-asyncoperation-resource-format)
- [Long running operations running more than a day](#long-running-operations-running-more-than-a-day)

## Asynchronous Operations ##

Some REST operations can take a long time to complete. Although REST is not supposed to be stateful, some operations are made asynchronous while waiting for the state machine to create the resources, and will reply before the operation on resources are completed. For such operations, the following guidance applies.

**Please note:**

    * Retry-After should be (integer), will not support http-date.
    * Location header should be absolute URI (partial URI is not supported). 
    * Azure-AsyncOperation header should be absolute URI (partial URI is not supported).
    * Location URI should return 202 Accepted with no response body while operation is in progress.
    * Azure-AsyncOperation URI returns 200 and status of the operation is present in the response payload. The response should match the RPC contract.

## Creating or Updating Resources Asynchronously ##

### Creating/Updating using PUT ###

The API flow for PUT should be to:

1. Respond to the initial PUT request with a 201 Created or 200 OK (per normal guidance).
2. Since provisioning is not complete, the PUT response body **MUST** contain a provisioningState property set to a non-terminal value (as described [below](#provisioningstate-property)). Resource properties should reflect the update that is in progress (i.e. the state the resource will be in once the async operation is complete).
4. When provisioningState is non-terminal value, the response headers **MUST** include a Location header that points to a URL where the ongoing operation can be monitored (as described [below](#202-accepted-and-location-headers)) or/and a Azure-AsyncOperation header pointing to an Operation resource (as described [below](#azure-asyncoperation-resource-format)).
5. Future GETs on the resource that was created should continue to return a 200 Status Code and provisioningState property that is \*non-terminal\* as long as the provisioning is in progress. 
6. After the provisioning completes, the provisioningState property should transition to one of the terminal states. If an update to existing resource properties failed then those properties should be reverted to their previous state if that best represents the end state of the resource after the failed operation.
7. The provisioningState property should be returned on all future GETs, even after it is complete, until some other operation (e.g. a DELETE or UPDATE) causes it to transition to a non-terminal state.

<br/>

  Some of the resource provider's existing PUT APIs return 202 Accepted. Please note this is a old model. For PUT requests, 201 / 200 + ProvisioningState is the preferred model. For more details, refer Microsoft API guidelines on [long running operations](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#131-resource-based-long-running-operations-relo).


### Updating using PATCH ###

The API flow for PATCH on an existing resource should be to:

1. Respond to the initial PATCH request with a 202 Accepted
2. The response headers **MUST** include a Location header that points to a URL where the ongoing operation can be monitored (as described [below](#202-accepted-and-location-headers)).
3. **Optional:** The response headers may include an Azure-AsyncOperation header pointing to an Operation resource (as described [below](#azure-asyncoperation-resource-format)).
4. If a provisioningState property is used for the resource, it **MUST** transition to a non-terminal state like &quot;Updating&quot;
5. If the PATCH completes successfully, the URL that was returned in the Location header **MUST** now return what would have been a successful response if the API completed (e.g. a response body / header / status code).

## Delete Resource Asynchronously ##

The API flow should be to:

1. Respond to the initial DELETE request with a 202 Accepted
2. The response headers **MUST** include a Location header that points to a URL where the ongoing operation can be monitored (as described [below](#202-accepted-and-location-headers)).
3. **Optional:** The response headers may include an Azure-AsyncOperation header pointing to an Operation resource (as described [below](#azure-asyncoperation-resource-format)).
4. If a provisioningState property is used for the resource, it **MUST** transition to a non-terminal state like &quot;Deleting&quot;
5. If the DELETE completes successfully, the URL that was returned in the Location header **MUST** now return a 200 OK or 204 NoContent to indicate success and the resource **MUST** disappear.

## Call Action POST Asynchronously ##

The API flow for POST {resourceUrl}/{action} should be:

1. Respond to the initial POST request with a 202 Accepted
2. The response headers **MUST** include a Location header that points to a URL where the ongoing operation can be monitored (as described [below](#202-accepted-and-location-headers)).
3. **Optional:** The response headers may include an Azure-AsyncOperation header pointing to an Operation resource (as described [below](#azure-asyncoperation-resource-format)).
4. If the POST completes successfully, the URL that was returned in the Location header **MUST** now return what would have been a successful response if the API completed (e.g. a response body / header / status code). It is acceptable for this to be a 200/204 NoContent if the action does not require a response (e.g. restarting a VM).

## ProvisioningState property ##

The provisioningState property has only three terminal states: **Succeeded** , **Failed** and **Canceled**. If the resource returns no provisioningState, it is assumed to be **Succeeded**.

Any state other than above three terminal states is a non-terminal state (e.g. "PreparingVMDisk", "MountingDrives", "SelectingHosts", "Deleted", "Accepted" etc.). Each individual RP is able to define their own non-terminal / transitioning / ephemeral states that are set before the resource reaches terminal states.

The basic functionality of the template orchestration engine and other clients is to:

1. PUT the resource to be created; and, then,
2. Poll at the resourceUri until the provisioningState enters a terminal state

before assuming that the resource creation has completed. In the case of the template orchestration engine, it would not create dependent resources until the resource creation is completed.

An example of the response body can be seen below:

**Response Body**

    {
      "location": "North US",
      "tags": {
        "key1": "value 1",
        "key2": "value 2"
      },

      "properties": {
        "provisioningState": "Succeeded",
        "comment": "Resource defined structure"
      },
    }

If the user does a PUT with provisioningState (e.g. after doing a GET), the resource provider should treat the field as a normal readonly property.

If the user provides a provisioningState matching the value \*on the server\*, the server should ignore it. If the user provides a provisioningState that does \*not\* match the value on the server, the resource provider should reject the call as a 400 BadRequest.

## 202 Accepted and Location Headers ##

The asynchronous operation APIs frequently use the 202 Accepted status code and Location headers to indicate progress. The flow is described below:

1. Return HTTP code 202 (Accepted) with a Location header and, optionally, a Retry-After header. 
    - The 202 Accepted should return no response body.
    - The URI in the location header should be a full absolute URI and a public facing URI.
    - The URI host must be the same as the host in the referrer header. 
    - The time interval in the Retry-After header can only be specified in seconds, with a minimum of 10 seconds and a maximum of 10 minutes.
2. Clients invoke the URI specified in the Location header using the GET verb.
3. Clients should wait for the Retry-After interval, if it was specified, or the default of 60 seconds if it was not, before polling again.
4. If the operation is not complete, return 202 (Accepted) again with Location header and optionally an updated Retry-After header.
5. If the operation is complete, return the **exact same response** that would have been returned had the operation been executed synchronously.
6. If the operation is not complete after maximum time, clients will give up and treat the operation as timed out.
7. If the 202 was returned in response to resource creation, the resource should \*not\* be visible until the async operation completes (i.e. GETs on the resource should return 404 and collection requests should not include the resource). If the resource should be visible, the provisioningState pattern should be used.

The Uri for the Location header can be either underneath a subscription level operations URL:

Examples:
https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/providers/{namespace}/locations/westus/operationResults/{operationId}?api-version={api-version} <br/>

https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/providers/{namespace}/operationResults/{operationId}?api-version={api-version}

or underneath the resource on which the operation was invoked:

Examples:
https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/providers/{namespace}/{resourceType}/{resourceName}/operationResults/{operationId}?api-version={api-version} <br/>

https://&lt;endpoint&gt;/subscriptions/{subscriptionId}/providers/{namespace}/{resourceType}/{resourceName}/{childResourceType}/{childResourceName}/operationResults/{operationId}?api-version={api-version}

The _endpoint_ can be determined by the hostname of the URI in the _referer_ header.

Location URI under subscription scope is preferred. If operationResults is exposed under resource scope, then for long running operations like delete, operationResults resource will not be accessible when the resource gets deleted.

### Example flow ###
| Clients | Resource Provider |
| --- | --- |
| DELETE /…/subscriptions/sub1/providers/Microsoft.Test/tests/test1 |   |
|   | 202 Accepted <br/> Location: /…/subscriptions/sub1/providers/Microsoft.Test/operationResults/id1 <br/> Retry-After: 60 |
| Waits 60 seconds |   |
| GET /…/subscriptions/sub1/providers/Microsoft.Test/operationResults/id1  |   |
|   | 202 Accepted <br/> Location: /…/subscriptions/sub1/providers/Microsoft.Test/operationResults/id1 <br/> Retry-After: 10 |
| Waits 10 seconds |   |
| GET /…/subscriptions/sub1/providers/Microsoft.Test/operationResults/id1 |     |
|   | 204 NOCONTENT |

## Azure-AsyncOperation Resource format ##
The operation resource format returned by the Azure-AsyncOperation header is as follows-

    {
      "id": "/subscriptions/id/locations/westus/operationsStatuses/sameguid",
      "name": "sameguid",
      "status" : "RP defined values | Succeeded | Failed | Canceled",
      "startTime": "<DateLiteral per ISO8601>",
      "endTime": "<DateLiteral per ISO8601>",
      "percentComplete": <double between 0 and 100>
      "properties": {
        /\* The resource provider can choose the values here, but it should only be
            returned on a successful operation (status being "Succeeded"). \*/
      },
      "error" : {
        /\* RP must return the *code* and *messages* fields. Please use the schema for the "ErrorResponse" Type from the Common Types definition in the Azure Rest API Specifications [repository](https://github.com/Azure/azure-rest-api-specs/blob/master/specification/common-types/resource-management/v2/types.json) \*/
        "code": "BadArgument",
        "message": "The provided database &#39;foo&#39; has an invalid username."
      }
    }

**The operation resource should be used for Azure-AsyncOperation header**. Location header should have different behavior described above.

The status values are similar to provisioningState – the transient / intermediate values can be service specific; however, the terminal values can be one of (1) Succeeded, (2) Failed or (3) Canceled. All other values imply that the operation is still running / being applied.

When reading the operation resource entity, the status code should always be 200. The response payload indicates if the async operation is still running or has completed (successfully or with a failure).

If the response status code is a 4xx or 5xx value, it indicates a failure in reading the _status_ but \*not\* in the underlying operation.

| Element name | Description |
| --- | --- |
| id | **Optional.** (but recommended). It should match what is used to GET the operation resource. |
| name | **Optional.** (but recommended). It must match the last segment of the &quot;id&quot; field, and will typically be a GUID / system generated value. |
| status | **Required.** … |
| properties | **Optional** Like the properites envelope for resources, the resource provider can choose the values here. However, it should only be set if the status is Succeeded. |
| error | **Required if status == failed or status == canceled.** This is the OData v4 error format, used by the RPC and will go into the v2.2 Azure REST API guidelines.  The full set of optional properties (e.g. inner errors / details) can be found in the &quot;Error Response&quot; section. |
| error.code | **Required if status == failed or status == canceled.**  If status == failed, provide an invariant error code used for error troubleshooting, aggregation, and analysis. |
| error.message | **Required if status == failed.**   **Localized.**  If status == failed, provide an actionable error message indicating what error occurred, and what the user can do to address the issue. |
| startTime | **Optional.**  DateLiteral using ISO8601, per 1API guidelines. |
| endTime | **Optional.**  DateLiteral using ISO8601, per 1API guidelines. |
| percentComplete | **Optional.**  Double between 0 and 100. |

## Long running operations running more than a day ##

Some long running operations may take more than a day to complete. In addition to following the above mentioned async guidelines, Resource Providers can choose to expose a new proxy resource for retrieving the status of the operation. If the long running operation is initiated from a client like Portal, the Location or Azure-AsyncOperation URI will be lost if user closes the portal. Client can do a GET on the proxy resource to retrieve the latest status of the operation anytime.

For example, execute tests is a long running operation taking more than a day
POST /…/subscriptions/subId/resourceGroups/rgId/providers/Microsoft.Test/scenarioTests/test1/execute

The latest test execution status is retrieved doing GET on the testExecutionResults proxy resource
GET /…/subscriptions/subId/resourceGroups/rgId/providers/Microsoft.Test/scenarioTests/test1/testExecutionResults/latest
