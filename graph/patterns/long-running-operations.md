# Long running operations

Microsoft Graph API Design Pattern

*The long running operations (LRO) pattern provides the ability to model operations where processing a client request takes a long time, but the client isn't blocked and can do some other work until operation completion.*

## Problem

The API design requires modeling operations on resources, which takes a long time
to complete, so that API clients don't need to wait and can continue doing other
work while waiting for the final operation results. The client should be able to
monitor the progress of the operation and have an ability to cancel it if
needed.

The API needs to provide a mechanism to track the work
being done in the background. The mechanism needs to be expressed in the same
web style as other interactive APIs. It also needs to support checking on the status and/or
being notified asynchronously of the results.

## Solution

The solution is to model the API as a synchronous service that returns a
resource that represents the eventual completion or failure of a long running
operation.

There are two flavors of this solution:

- The returned resource is the targeted resource and includes the status of
  the operation. This pattern is often called RELO (resource-based long running operation).

<!-- markdownlint-disable MD033 -->
<p align="center">
  <img src="RELO.gif" alt="The status monitor LRO flow"/>
</p>
<!-- markdownlint-enable MD033 -->

- The returned resource is a new API resource called *stepwise operation* and is created to track the status. This LRO solution is similar to the concept of Promises or Futures in other programming languages.

<!-- markdownlint-disable MD033 -->
<p align="center">
  <img src="LRO.gif" alt="The status monitor LRO flow"/>
</p>
<!-- markdownlint-enable MD033 -->

When in doubt, the RELO pattern is the preferred pattern for long running operations. There is quite a bit of inconsistency in how stepwise operations have been implemented, and tooling/SDK support is suboptimal. RELO also provides a lot more flexibility and power in terms of being able to represent progress in a more meaningful way than a simple percentage.

In general, Microsoft Graph API guidelines for long running operations follow [Microsoft REST API
Guidelines](https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#13-long-running-operations).
There are some deviations from the base guidelines where Microsoft Graph API standards require that you do one of the following:

- For the RELO pattern, you should return the Location header that indicates the location of the resource.
  - The API response says the targeted resource is being created by returning a 201 status code and the resource URI is provided in the Location header, but the response indicates that the request is not completed by including "Provisioning" status.

- For the LRO pattern, you should return the Location header that indicates the location of a new stepwise operation resource.
  - The API response says the operation resource is being created at the URL provided in the Location header and indicates that the request is not completed by including a 202 status code.
  - Microsoft Graph doesn’t allow tenant-wide operation resources; therefore, stepwise operations are often modeled as a navigation property on the target resource.

## When to use this pattern

Any API call that is expected to take longer than one second in the 99th percentile should be considered a 'long running operations' and use one of these patterns.

How do you select which pattern to use?

RELO relies on the concept of the resource having a state model (moving through various states of creating, created, deleting, deleted, updating, pending, publishing etc) and is suitable for modeling operations that drive the resource through those states. A common use case is 'provisioning' flows where you're trying to create something and it takes a while for it to reach usable state. In those cases you immediately create an actual entity that has an ID and return it, and the client polls to find out when it becomes operational. Other likely scenarios include "backup", where the operation is expected to take hours/days/months, and an admin might want to track progress by periodically looking at a portal.

'Stepwise' is more useful for cases where the you're doing something 'slow', but the entity you are operating on does not have a formal 'state' model or the operation does not drive the entity into a new 'state'. Let's look at a hypothetical API like 'copy folder' - the original folder isn't modified, and it doesn't have the concept of a 'state' that can change. But there is an async job slowly working in the background to complete the action you requested, and you might want to track progress/completion of the operation.

In those cases, you could theoretically create an entire model to represent a "folder copy job", define an enum for its lifecycle stages, and use RELO to track the lifecycle of the job from start to completion. However, it's pretty heavyweight to create a new entity and enum just to answer the question "Is it done?". This is especially true if the operation is expected to only take a few seconds to complete. In these cases, Stepwise is simpler - just return a URL that you can poll for status, or that you can ignore for 'fire and forget'.

However, let's say the copy operation is often expected to take months to complete, and someone is likely to want to periodically open up some management UX to see its progress and potentially cancel or retry in case of a few files/subfolders failing to copy. In this case it makes much more sense to model the copy operation as an explicit entity with rich management capabilities, at which point RELO makes more sense.


Stepwise can be thought of as a special case of RELO that is 'wrapped' to make simpler. The monitor/polling URL is just the Graph URL of an 'operation' entity that has its own state model (in progress, complete etc). The main distinction is that:

- 'operation' has a standardized schema across all APIs using the stepwise pattern (sort of - various workloads have implemented slight variations) whereas in RELO there is no standardization. This has both pros (simple, consistent) and cons (functionality limited to 'lowest common denominator' of all stepwise scenarios).
- Stepwise hides the fact that there is an 'operation' entity. What we publicly document is that it is an opaque URL that is valid for some finite length of time. Although this is simple to comprehend, the returned URL is designed to be disposable/ephemeral which has both pros (fire and forget) and cons (hard to keep track of for anything longer than a few seconds to a minute).


Here are some heuristics for choosing between the two patterns:

1. If a service can create a resource with a minimal latency and continue updating its status according to the well-defined and stable state transition model until completion, then the RELO model is the best choice. This is often the case for provisioning and other resource lifecycle management scenarios.
2. For scenarios that involve a relatively short-duration 'fire and forget' operation that does not involve driving a resource through some state model, Stepwise is often preferred.
3. For scenarios that involve operations on the timescale of hours to months (backup, migration etc), the operation should be modeled explicitly as an entity and use RELO.
 
## Issues and considerations

- One or more API consumers MUST be able to monitor and operate on the same resource at the same time.

- The state of the system SHOULD always be discoverable and testable. Clients
    SHOULD be able to determine the system state even if the operation tracking
    resource is no longer active. Clients MAY issue a GET on some resource to
    determine the state of a long running operation.

- The long running operations pattern SHOULD work for clients looking to "fire and forget"
    and for clients looking to actively monitor and act upon results.

- The long running operations pattern might be supplemented by the [change notification pattern](./change-notification.md).

- Cancellation of a long running operation does not explicitly mean a rollback. On a per API-defined case, it
    might mean a rollback or compensation or completion or partial completion,
    etc. Following a canceled operation, the API should return a consistent state that allows
    continued service.

- A recommended minimum retention time for a stepwise operation is 24 hours.
    Operations SHOULD transition to "tombstone" for an additional period of time
    prior to being purged from the system.
    
- Services that provide a new operation resource MUST support GET semantics on the operation.
- Services that return a new operation MUST always return an LRO (even if the LRO is created in the completed state); that way API consumers don't have to deal with two different shapes of response.

## Examples

### Create a new resource using RELO

A client wants to provision a new database:

```
POST https://graph.microsoft.com/v1.0/storage/databases/

{
"displayName": "Retail DB",
}
```

The API responds synchronously that the database has been created and indicates
that the provisioning operation is not fully completed by including the
Content-Location header and status property in the response payload:

```
HTTP/1.1 201 Created
Location: https://graph.microsoft.com/v1.0/storage/databases/db1

{
"id": "db1",
"displayName": "Retail DB",
"status": "provisioning",
[ … other fields for "database" …]
}
```

The client waits for a period of time, and then invokes another request to try to get the database status:

```
GET https://graph.microsoft.com/v1.0/storage/databases/db1

HTTP/1.1 200 Ok
{
"id": "db1",
"displayName": "Retail DB",
"status": "succeeded",
[ … other fields for "database" …]
}
```

### Cancel RELO operation

A client wants to cancel provisioning of a new database:

```
DELETE https://graph.microsoft.com/v1.0/storage/databases/db1

```

The API responds synchronously that the database is being deleted and indicates
that the operation is accepted and is not fully completed by including the
status property in the response payload. The API might provide a
recommendation to wait for 30 seconds:

```
HTTP/1.1 202 Accepted
Retry-After: 30

{
"id": "db1",
"displayName": "Retail DB",
"status": "deleting",
[ … other fields for "database" …]
}
```

The client waits for a period of time, and then invokes another request to try to get the deletion status:

```
GET https://graph.microsoft.com/v1.0/storage/databases/db1

HTTP/1.1 404 Not Found
```
### Create a new resource using the stepwise operation

```
POST https://graph.microsoft.com/v1.0/storage/archives/

{
"displayName": "Image Archive",
...
}
```

The API responds synchronously that the request has been accepted and includes
the Location header with an operation resource for further polling:

```
HTTP/1.1 202 Accepted

Location: https://graph.microsoft.com/v1.0/storage/operations/123

```

### Poll on a stepwise operation

```

GET https://graph.microsoft.com/v1.0/storage/operations/123
```

The server responds that results are still not ready and optionally provides a
recommendation to wait 30 seconds:

```
HTTP/1.1 200 OK
Retry-After: 30

{
"createdDateTime": "2015-06-19T12-01-03.4Z",
"lastActionDateTime": "2015-06-19T12-01-03.45Z",
"status": "running"
}
```

The client waits the recommended 30 seconds and then invokes another request to get
the results of the operation:

```
GET https://graph.microsoft.com/v1.0/storage/operations/123
```


The server responds with a "status:succeeded" operation that includes the resource
location:

```
HTTP/1.1 200 OK

{
"createdDateTime": "2015-06-19T12-01-03.45Z",
"lastActionDateTime": "2015-06-19T12-06-03.0024Z",
"status": "succeeded",
"resourceLocation": "https://graph.microsoft.com/v1.0/storage/archives/987"
}
```

### Trigger a long running action using the stepwise operation

```
POST https://graph.microsoft.com/v1.0/storage/copyArchive

{
"displayName": "Image Archive",
"destination": "Second-tier storage"
...
}
```

The API responds synchronously that the request has been accepted and includes
the Location header with an operation resource for further polling:

```
HTTP/1.1 202 Accepted

Location: https://graph.microsoft.com/v1.0/storage/operations/123

```
