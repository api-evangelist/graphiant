---
name: Triage Graphiant alarms and route notifications
description: Read parent and child alarms, acknowledge them, suppress noise with allowlists and mute lists, and wire notification rules to email or an outbound integration.
api: openapi/graphiant-portal-openapi-original.json
generated: '2026-08-01'
method: generated
operations:
  - POST /v2/parentalertlist
  - POST /v2/childalertlist
  - POST /v2/ack/createupdate
  - POST /v2/rulelist
  - POST /v2/rule/enabledisable
  - POST /v2/allowlist/create
  - GET /v2/allowlist-by-enterprise
  - POST /v2/notificationmutelist/create
  - POST /v2/notification/create
  - POST /v2/notificationlist
  - POST /v2/notification/enabledisable
  - POST /v2/integration/
  - GET /v2/integration/getall/{enterpriseId}
  - GET /v2/integration/test/{enterpriseId}/{integrationId}
---

# Triage Graphiant alarms and route notifications

Complete `graphiant-authenticate-and-scope.md` first. Reading alarms needs
`monitoringAndTroubleshooting`; changing rules or integrations needs `read_write` on it.

The alarm surface is the `/v2` API. Note that most reads here are **POST** operations —
the filter travels in the request body. A `POST` to `/v2/parentalertlist` is a query, not
a mutation.

## 1. Read the alarms

- `POST /v2/parentalertlist` — parent alarms for the enterprise
- `POST /v2/childalertlist` — the child alarms aggregated under a parent

Parents aggregate children. Triage at the parent level; drill into children only for the
specific fault.

## 2. Acknowledge

`POST /v2/ack/createupdate` — record an acknowledgement against an alert. Acknowledge
before you start remediating so a second operator does not duplicate the work.

## 3. Suppress noise

Two distinct mechanisms — pick the right one:

| Mechanism | Effect | Create | Read | Delete |
|---|---|---|---|---|
| **Allowlist** | the condition stops being treated as an alarm | `POST /v2/allowlist/create` | `GET /v2/allowlist/{ruleId}`, `GET /v2/allowlist-by-enterprise` | `DELETE /v2/allowlist/deletebyalertid/{alertId}`, `DELETE /v2/allowlist/deletebyentityid/{entityId}` |
| **Notification mute list** | the alarm still raises, but stops notifying | `POST /v2/notificationmutelist/create` | `GET /v2/notificationmutelist/{ruleId}` | `DELETE /v2/notificationmutelist/deletebyalertid/{alertId}`, `DELETE /v2/notificationmutelist/deletebyentityid/{entityId}` |

Prefer the **mute list** for temporary noise during a maintenance window — the alarm
history stays intact. Use the **allowlist** only for a condition that is genuinely
expected in this environment.

Both delete surfaces work by `alertId` *or* by `entityId`. Deleting by `entityId`
removes every suppression for that device or object at once — check before you use it.

## 4. Manage rules

- `POST /v2/rulelist` — list alarm rules
- `POST /v2/rule/enabledisable` — toggle a rule
- `POST /v2/notification/create`, `/update`, `/delete`, `/enabledisable` — notification
  routing rules
- `POST /v2/notificationlist` — list them
- `POST /v2/aggregated-notification/enable-disable`, `GET /v2/aggregated-notification/get-state`
  — control alarm aggregation (roll many alarms into one notification)

A notification rule carries recipients, a free-form message appended to the alarm, and
optionally a target integration.

## 5. Route to an outbound integration

- `GET /v2/integration/getall/{enterpriseId}` — what is configured
- `POST /v2/integration/` — create
- `PUT /v2/integration/{integrationId}` / `DELETE /v2/integration/{integrationId}`
- `GET /v2/integration/test/{enterpriseId}/{integrationId}` — send a test notification

Supported destinations: **Microsoft Teams, Atlassian OpsGenie, PagerDuty, OpsRamp**, and
a **generic webhook** for anything else (the docs name Slack and Rootly as examples). A
generic webhook integration takes a nickname and a routing key — the incoming API key of
the destination app.

**Always fire the test operation after creating an integration.** Graphiant publishes no
payload schema, no signature scheme, and no retry or dead-letter policy for the outbound
webhook, so a silent misconfiguration will not surface until a real alarm is lost.

## Caveats

- There is **no inbound event stream** — no AsyncAPI, no SSE, no WebSocket. Either poll
  the alert lists or receive pushes at a configured integration.
- No pagination on any list operation.
- Errors use `{errorCode, displayError, detailedError}`.

See `asyncapi/graphiant-notifications-webhooks.yml`.
