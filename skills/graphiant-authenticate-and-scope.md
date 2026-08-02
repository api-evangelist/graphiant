---
name: Authenticate against the Graphiant Portal API and establish tenant scope
description: Obtain a bearer token, discover the caller's permission matrix, and (for MSP users) switch into the correct enterprise tenant before doing any other work.
api: openapi/graphiant-portal-openapi-original.json
generated: '2026-08-01'
method: generated
operations:
  - POST /v1/auth/login
  - GET /v1/auth/user
  - GET /v1/enterprises
  - GET /v1/users/{id}/enterprises
  - GET /v1/auth/session
  - GET /v1/auth/session/root
  - GET /v1/auth/refresh
---

# Authenticate and establish tenant scope

Every other Graphiant skill depends on this one. Do not skip the scope check: an MSP
user's token defaults to the MSP's own enterprise, so a device list taken before
switching context will silently return the wrong tenant's devices.

## Base

`https://api.graphiant.com` — the host returns `403 Forbidden` to every anonymous
request, so there is nothing to probe before you hold a token.

## 1. Get a token

`POST /v1/auth/login`

```json
{ "username": "<user>", "password": "<password>" }
```

Response:

```json
{ "auth": true, "accountType": "enterprise", "token": "gr-auth-…" }
```

Send it on every subsequent call as `authorization: Bearer <token>`.

**Send exactly one Authorization header.** When using the generated SDKs, set the
per-call `authorization` argument *or* `Configuration.api_key` for `jwtAuth` — never
both. Two headers make some upstream gateways return `400`.

`accountType` is one of `msp`, `enterprise`, `graphiant`. Branch on it in step 3.

## 2. Read your own permissions

`GET /v1/auth/user`

Returns `userId`, the active `enterpriseId`, and a `permissions` object with 18 domains
(`assetManager`, `networkConfiguration`, `servicePolicies`, `safetyAndSecurity`,
`globalServices`, `userAndTenantManagement`, `insights`, `reports`,
`monitoringAndTroubleshooting`, `compliance`, `logs`, `developerTools`, `licensing`,
`billingAndInvoicing`, `orderStatus`, `support`, `gateway`, `b2b`), each graded `read`
or `read_write`.

Gate your own behaviour on this. If `networkConfiguration` is `read`, do not attempt a
configuration push — the API will reject it, but you should not have tried.

## 3. Establish tenant scope (MSP callers only)

If `accountType` is `msp`, the token starts in the MSP's own enterprise.

- `GET /v1/enterprises` — the tenants visible to this token
- `GET /v1/users/{id}/enterprises` — the tenants this specific user may switch into
  (association is explicit; MSP admin status alone does not grant it)
- `GET /v1/auth/session?enterpriseId=<id>` — switch context into a tenant
- `GET /v1/auth/session/root` — switch back to the MSP's own enterprise

Confirm the switch with `GET /v1/auth/user` and check `enterpriseId` before continuing.

**Context lives on the token, not on the request.** One token cannot serve two tenants
concurrently. If you need parallel work across tenants, hold one token per tenant.

## 4. Keep the token alive

Tokens are valid for **30 minutes**. Refresh *before* expiry:

`GET /v1/auth/refresh` — accepts the current or the just-expired token and returns a
new one.

Refresh proactively on a timer. Do not wait for a failure: a 403 mid-workflow may
arrive between a configuration submit and the job poll that tells you whether it landed.

## Errors

Graphiant does not use RFC 9457. Errors arrive as
`{"errorCode": 403, "displayError": "Token Expired", "detailedError": ""}`.

Branch on `displayError`:

| displayError | Meaning | Action |
|---|---|---|
| `Token Expired` | past 30 minutes | `GET /v1/auth/refresh` |
| `Invalid Token` | session revoked | log in again |
| `User logged out` | session revoked | log in again |

`422 Unprocessable Entity — GRPC error` on the session endpoints means a backend call
failed for a well-formed request. Retry once, then stop; reshaping the request will not
help.

See `authentication/graphiant-authentication.yml` and
`errors/graphiant-problem-types.yml`.
