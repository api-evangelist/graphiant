---
name: Inventory a Graphiant network
description: Build a read-only picture of an enterprise's Graphiant estate — devices, sites, regions, LAN segments, gateways and their health — without changing anything.
api: openapi/graphiant-portal-openapi-original.json
generated: '2026-08-01'
method: generated
operations:
  - GET /v1/edges-summary
  - GET /v1/devices/{deviceId}
  - GET /v1/devices/{deviceId}/interfaces
  - GET /v1/sites
  - GET /v1/sites/map/details
  - GET /v1/regions
  - GET /v1/lan-segments
  - GET /v1/gateways/summary
  - POST /v1/backbone-health/overview
---

# Inventory a Graphiant network

Read-only. Safe to run unattended. Complete
`graphiant-authenticate-and-scope.md` first — and if the caller is an MSP, confirm you
are in the intended tenant before you start counting devices.

## 1. Devices

`GET /v1/edges-summary` is the single most useful call in the API. It returns every
device with:

| Field | Why it matters |
|---|---|
| `deviceId` | the key for every per-device call |
| `hostname`, `serialNum`, `model` | identity |
| `role` (`cpe`, …), `status` (`active`, …) | placement and state |
| `portalStatus` | must be `Ready` before any config change |
| `ttConnCount` | tunnel-terminator connections; must be `2` before any config change |
| `swVersion` / `swName` | e.g. `2506.202506021956` / `25.6.1` |
| `site`, `siteId`, `region`, `overrideRegion` | topology placement |
| `location`, `discoveredLocation`, `ipDetected` | geo and WAN address |
| `lastBootedAt`, `firstAppearedOn` | protobuf `{seconds, nanos}` timestamps, UTC |

Filter with bracketed query parameters:

```
GET /v1/edges-summary?filter[role]=UnknownDeviceRole&filter[status]=active
```

There is **no pagination** anywhere in this API. The full collection comes back in one
response — size your buffers accordingly on large estates.

Per-device detail: `GET /v1/devices/{deviceId}` and
`GET /v1/devices/{deviceId}/interfaces`.

## 2. Topology

- `GET /v1/sites` — enterprise sites
- `GET /v1/sites/map/details` — sites with per-site device summaries and LAN-segment
  buckets; the best single call for a topology view
- `GET /v1/regions` — Graphiant backbone regions

## 3. Connectivity

- `GET /v1/lan-segments` — named L3 segments (the VRF-like construct) with their
  networks, static routes, BGP neighbours, DHCP subnets, syslog targets and IPFIX
  exporters
- `GET /v1/gateways/summary` — gateway services. Note the read is on `/summary`;
  `/v1/gateways` itself exposes only `POST`, `DELETE` and `PUT`.

## 4. Health

`POST /v1/backbone-health/overview` — a read expressed as POST because the filter
travels in the body. Several Graphiant reads are shaped this way; treat a POST to a
`*list`, `*summary` or `*overview` path as a query, not a mutation.

## Timestamps

Time fields are protobuf `Timestamp` objects, not RFC 3339 strings:

```json
"lastBootedAt": { "seconds": 1750202669, "nanos": 537864000 }
```

Convert from `seconds` (Unix epoch) yourself, and label the result UTC — that is what
the Graphiant CLI does.

## Identifiers

`deviceId`, `enterpriseId`, `siteId` are opaque 64-bit integers with no type prefix.
They occupy visibly different numeric ranges, but Graphiant publishes no prefix scheme.
**Never infer entity type from an id value** — the path tells you the type, the id does
not. See `data-model/graphiant-data-model.yml`.
