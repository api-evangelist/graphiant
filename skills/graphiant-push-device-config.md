---
name: Push a Graphiant device configuration safely
description: Validate device readiness, submit a declarative configuration fragment, and resolve the returned job to confirm the change actually applied. This skill mutates production network state.
api: openapi/graphiant-portal-openapi-original.json
generated: '2026-08-01'
method: generated
risk: high
operations:
  - GET /v1/edges-summary
  - GET /v1/devices/{deviceId}/interfaces
  - PUT /v1/devices/{deviceId}/config
  - GET /v1/devices/{deviceId}/jobs/{jobId}
  - GET /v1/devices/{deviceId}/draft
  - POST /v1/devices/{deviceId}/draft
  - DELETE /v1/devices/{deviceId}/draft
---

# Push a Graphiant device configuration safely

**This skill changes production network configuration.** It has no idempotency key, it
is asynchronous, and a `null` value means *delete*. Require explicit human approval
before the `PUT` in step 3. Do not run this unattended.

Complete `graphiant-authenticate-and-scope.md` first and confirm
`permissions.networkConfiguration == "read_write"`.

## 1. Preflight — the device must be ready

`GET /v1/edges-summary`, find your device, and assert **both**:

- `portalStatus == "Ready"` (the device is in sync with the portal)
- `ttConnCount == 2` (the device has both tunnel-terminator connections to Graphiant)

If either check fails, stop. Graphiant documents this as a required precondition. A push
to a device that is not Ready will not land, and you will not get a synchronous error
telling you so.

## 2. Read the current shape

`GET /v1/devices/{deviceId}/interfaces` to see what exists before you change it.

## 3. Submit the desired state

`PUT /v1/devices/{deviceId}/config`

The body is a **partial desired-state document**, not a full replacement:

```json
{
  "edge": {
    "interfaces": {
      "GigabitEthernet8/0/0": {
        "interface": {
          "subinterfaces": {
            "18": {
              "interface": {
                "vlan": 18,
                "lan": "lan-7-test",
                "alias": "non_production",
                "description": "lan-7",
                "adminStatus": true,
                "ipv4": { "address": { "address": "10.2.7.1/24" } },
                "ipv6": { "address": { "address": "2001:10:2:7::1/64" } }
              }
            }
          }
        }
      }
    },
    "segments": {
      "lan-7-test": {
        "networks": [], "bgpRedistribution": {}, "bgpNeighbors": {},
        "syslogTargets": {}, "staticRoutes": {}, "dhcpSubnets": {},
        "bgpAggregations": {}, "ipfixExporters": {}
      }
    }
  },
  "description": "adding a subinterface",
  "configurationMetadata": { "name": "" }
}
```

### The two semantics you must not confuse

| You write | Graphiant does |
|---|---|
| key **omitted** | leave the existing value untouched |
| key set to **`null`** | **delete that element** |

Deleting subinterface 18:

```json
{"edge":{"interfaces":{"GigabitEthernet8/0/0":{"interface":{"subinterfaces":{"18":{"interface":null}}}}}},
 "description":"deleting a subinterface","configurationMetadata":{"name":""}}
```

A serialiser that emits absent optional fields as `null` will silently delete
configuration. Serialise omissions as *omissions*.

Always set a meaningful `description` — it is recorded against the job and is the only
audit trail a later reader gets.

Circuits are referenced from an interface **by name** (`"circuit": "c-gigabitethernet5-0-0"`)
and LAN segments **by name** (`"lan": "lan-7-test"`), not by id. The referenced object
must exist or be created in the same document.

## 4. Resolve the job — the response is not the outcome

The `PUT` returns HTTP 200 with:

```json
{ "jobId": 1552067 }
```

**200 means accepted, not applied.** Poll:

`GET /v1/devices/{deviceId}/jobs/{jobId}`

until it reaches a terminal state. Report the job outcome, never the HTTP status, as the
result of the change.

## 5. Retry discipline

There is **no `Idempotency-Key`** on this endpoint. Replaying the same desired-state
document converges to the same device state — that is a property of the declarative
model, not a provider guarantee — but each submission creates a *new job*.

Therefore: **never blind-retry a config PUT.** On a timeout or a 5xx, first list or poll
for a job that may already exist, and only resubmit if none does.

## Drafts

`/v1/devices/{deviceId}/draft` supports `GET`, `POST` and `DELETE`. Where a review step
is wanted, stage into a draft rather than pushing straight to `config`.

## Errors

`{"errorCode": …, "displayError": …, "detailedError": …}`. A `403` with
`displayError: "Token Expired"` mid-workflow is the dangerous case — the submit may have
succeeded before the token lapsed. Refresh (`GET /v1/auth/refresh`) and **poll for the
job**; do not resubmit.

See `conventions/graphiant-conventions.yml` and `errors/graphiant-problem-types.yml`.
