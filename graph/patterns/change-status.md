# Change Status & Action Request

Microsoft Graph API Design Pattern

_The change status pattern is an effective alternative to model the ability to trigger side effects while keeping an agnostic, data driven approach_

## Problem

In some cases, exposed resources can trigger side effects that modify the system behavior. For instance, a [`riskyUser`](https://docs.microsoft.com/en-us/graph/api/resources/riskyuser?view=graph-rest-1.0) can be [`confirmedCompromised`](https://docs.microsoft.com/en-us/graph/api/riskyuser-confirmcompromised?view=graph-rest-1.0&tabs=http) or [`dismissed`](https://docs.microsoft.com/en-us/graph/api/riskyuser-confirmcompromised?view=graph-rest-1.0&tabs=http).

These situation are often modeled as a `POST` request to a specific endpoint in the same URI space of the resource, triggering the side effect directly and not returning any data. While it might be a quick and easy solution, there are some considerations:

- The API should be exposing resources and allow their manipulation, while Actions resemble a remote procedure call on an object, typical of the [OOP][1]
- It couples the request with the side effect, making it more difficult to test, log, replay and reason about what is happening in the system
- It forces the consumer to follow the [OOP][1] approach at the API level. We do not want to pursue any specific style and the developers writing the client might be working in other ways
- A `POST` request is used to create a resource record. Using it to do other operations through an action is confusing, and should really be a last resort when there are no clear alternatives.

## Solution

API Designers can prefer the usage of a property indicating the status and request for its modification or a datafyed version of the action that represents the **intent** of triggering a side effect rather than starting the execution directly through the HTTP request.

Taking in consideration the example of the `riskyUser` above, it is possible to keep the `riskState` property on the resource:

```xml
<EnumType Name="riskState">
    <Member Name="confirmedCompromised" Value="0"/>
    <Member Name="dismissed" Value="1"/>
    <Member Name="none" Value="2"/>
</EnumType>

<EntityType Name="riskyUser" >
    <Property Name="riskState" Type="riskState" />
</EntityType>
```

To confirm the user as compromised, we can send a `PATCH` request on the resource:

```bash
PATCH https://graph.microsoft.com/v1.0/identityProtection/riskyUsers/a57dc75f-24b5-47ce-b5e1-44822f5d4729

{"riskState": "confirmedCompromised"}
```

Instead of returning an empty body with `204` status code, depending on the situation, the response should contain the status of the request along with other properties modified because of the request, if it can be satisfied in a reasonable time:

```json
{
  "riskState": "confirmedCompromised",
  "riskDetail": "adminConfirmedUserCompromised",
  "riskLastUpdatedDateTime": "2015-06-19T12-01-03.4Z",
  "desiredRiskState": "confirmedCompromised"
}
```

If the response triggers a long running operation, the response should contain a reference to query its status, following the [existing guidance][2]:

```json
{
  "createdDateTime": "2015-06-19T12-01-03.4Z",
  "riskState": "confirmingCompromising"
}
```

The status on the resource shall stay the same until the long running operation is completed, but the workload can optionally return a property representing the desired status along with the current resource status to signal that it is working on a request:

`GET https://graph.microsoft.com/v1.0/identityProtection/riskyUsers/a57dc75f-24b5-47ce-b5e1-44822f5d4729`

```jsonc
{ "riskState": "none", "desiredRiskState": "confirmedCompromised" } // Other properties omitted for brevity
```

---

There might be cases where the change of the status might require some additional parameters, or where we might need to execute an action that is not 100% related to the entity. In such case, a `ChangeStatusRequest` resource might be created to accommodate this use case.

For instance, let's say the `riskyUser` confirmation process also requires a parameter that defines for how much time the user is going to be disabled. In this case, instead of a `riskyState` property on the exposed resource, we can define a new resource and create a new instance representing the intention:

```xml
<EnumType Name="riskState">
    <Member Name="confirmedCompromised" Value="0"/>
    <Member Name="dismissed" Value="1"/>
    <Member Name="unknown" Value="2"/>
</EnumType>

<EntityType Name="riskAssessment" >
    <Property Name="riskState" Type="riskState" />
    <Property Name="desiredRiskState" Type="riskState" />
    <Property Name="duration" Type="Edm.Duration" />
</EntityType>
```

```bash
POST https://graph.microsoft.com/v1.0/identityProtection/riskyUsers/a57dc75f-24b5-47ce-b5e1-44822f5d4729/riskAssessments

{"riskState": "confirmedCompromised", "duration": 3600}

HTTP 200/OK

{
  "riskState": "confirmedCompromised",
  "riskDetail": "adminConfirmedUserCompromised",
  "riskLastUpdatedDateTime": "2015-06-19T12-01-03.4Z",
  "desiredRiskState": "confirmedCompromised"
}
```

## Issues and Considerations

The nature of this pattern is to privilege resources and data manipulation rather than taking an opinionated stand. This means that the generated SDK might look slightly different than the rest of the system, especially if autogenerated. On the other hand, it leaves a great degree of freedom to concretize an SDK according to the methodologies provided by the target programming language.

[1]: https://en.wikipedia.org/wiki/Object-oriented_programming
[2]: https://github.com/microsoft/api-guidelines/blob/vNext/Guidelines.md#13-long-running-operations