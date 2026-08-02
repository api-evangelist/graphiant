---
name: Plan and schedule a Graphiant software rollout
description: Read the current software state of an estate, understand which release train each device is on, and schedule an upgrade rollout against a device set.
api: openapi/graphiant-portal-openapi-original.json
generated: '2026-08-01'
method: generated
risk: high
operations:
  - GET /v1/edges-summary
  - GET /v1/software/rollouts
  - POST /v1/software/rollouts
  - PUT /v1/software/rollouts
  - GET /v1/software/rollouts/{id}
  - DELETE /v1/software/rollouts/{id}
  - POST /v1/software/rollouts/schedule
  - GET /v1/software/auto-upgrade/default
  - PUT /v1/software/auto-upgrade/default
  - GET /v1/software/releases/download
---

# Plan and schedule a Graphiant software rollout

**Upgrades reboot production network devices.** Creating or scheduling a rollout requires
explicit human approval. Reading the current state does not.

Complete `graphiant-authenticate-and-scope.md` first.

## 1. Establish where the estate sits today

`GET /v1/edges-summary` returns `swVersion` and `swName` per device — for example
`2506.202506021956` / `25.6.1`.

Graphiant names software `YY.MM.<counter>`: two-digit year, month, counter. Sort by that,
not lexically by `swVersion`.

## 2. Know which train you are moving to

Graphiant runs three concurrent release trains. Choose deliberately:

| Train | Released | Supported for | Use for |
|---|---|---|---|
| **Latest** | monthly | 2.5 months (1 month critical fixes, then 45 days spot fixes) | newest features, lab and pilot sites |
| **Recommended** | every 3 months (every 3rd Latest) | 6.5 months (3 months active critical fixes, then 3.5 months spot fixes) | the default for production |
| **Stable** | every 12 months (every 4th Recommended) | 12 months (6 months active critical fixes, then 6 months spot fixes) | long-lived sites you upgrade rarely |

A device left on a train past its support window stops receiving fixes and leaves QA
coverage. Support overlap with the preceding release is maintained so you can upgrade at
your convenience — but the window is finite.

Out-of-band **Maintenance Releases** are deployed when a fix cannot wait for the next
scheduled release; upgrade to those when advised.

`GET /v1/software/releases/download` lists what is available.

## 3. Read existing rollouts before creating another

- `GET /v1/software/rollouts` — current campaigns
- `GET /v1/software/rollouts/{id}` — one campaign

Check for an overlapping rollout targeting the same devices before you create one. There
is no idempotency key on the create operation and no server-side conflict check
documented — two overlapping rollouts against one device set is your problem to prevent.

## 4. Create and schedule

- `POST /v1/software/rollouts` — create a campaign against a device set
  (`UpgradeRolloutConfig`, `UpgradeRolloutDevice`)
- `PUT /v1/software/rollouts` — modify
- `POST /v1/software/rollouts/schedule` — attach a schedule; recurrence models are
  `UpgradeRecurringSchedule` with weekly, monthly or yearly variants
- `DELETE /v1/software/rollouts/{id}` — cancel

Schedule inside Graphiant's own maintenance posture where you can: **scheduled
maintenance is every fourth Friday of the month, 14:00–16:00 ET**, and that window counts
as *Available* under the SLA. Emergency maintenance counts as *Not Available*.

## 5. Auto-upgrade default

- `GET /v1/software/auto-upgrade/default` — read the enterprise default
- `PUT /v1/software/auto-upgrade/default` — change it

Changing the enterprise auto-upgrade default affects every device that does not override
it. Treat it as a higher-blast-radius change than any single rollout.

## SLA context

Graphiant commits to **>99.99% availability**, with service credits from 2% (99.90–99.98%)
up to 100% (below 92%). P1 incidents are acknowledged in 15 minutes with hourly updates
and a one-hour resolution target. A botched rollout that takes sites down is a P1 — know
the escalation path before you schedule.

Watch `https://status.graphiant.io/` during the window (note: `.io`, not `.com`).

See `lifecycle/graphiant-lifecycle.yml` and `changelog/graphiant-changelog.yml`.
