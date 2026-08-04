# Graphiant

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Graphiant is a Network-as-a-Service (NaaS) provider that replaces fragmented SD-WAN, MPLS
and cloud-networking stacks with a single cloud-native subscription service built on a
stateless core architecture. The platform connects branches, data centers, public and
neo-cloud environments, remote users, AI agents, applications, customers and business
partners, delivering private connectivity, end-to-end and post-quantum encryption,
data-sovereignty routing controls, Zero Trust Network Access, data loss prevention, deep
packet inspection and real-time observability.

- Website — https://www.graphiant.com/
- Documentation — https://docs.graphiant.com/
- API reference — https://docs.graphiant.com/apidocs
- Portal — https://portal.graphiant.com/
- GitHub — https://github.com/Graphiant-Inc
- Status — https://status.graphiant.io/
- Trust center — https://trust.graphiant.com/

## API

The **Graphiant Portal REST API** (`https://api.graphiant.com`) exposes 525 operations
across 460 paths and 1,568 component schemas, covering authentication and MSP tenant
switching, enterprises and users, edges, devices and device configuration jobs, sites,
circuits, interfaces and LAG, LAN segments and prefixes, BGP peering and routing filters,
gateways and public VIF, B2B extranet and Data Exchange, assurance and AI-adoption
analytics, alarms, notifications and integrations, licensing and billing.

The OpenAPI 3.0.0 bundle is published by Graphiant inside its SDK repositories
(`api/graphiant_api_docs_v26.7.0.json`) and is the source the Python and Go SDKs are
generated from. The API host itself returns `403` to every anonymous request, so the
contract is public only by way of those repositories.

**Authentication** is an opaque bearer token from `POST /v1/auth/login`, sent as
`authorization: Bearer <token>`, valid for 30 minutes and renewed at `/v1/auth/refresh`.
There is no OAuth 2.0 and no OIDC; authorization is an 18-domain per-user permission
matrix read from `GET /v1/auth/user`.

### Notable characteristics

- **Declarative device configuration** — `PUT /v1/devices/{deviceId}/config` takes a
  partial desired-state document and returns a `jobId`. HTTP 200 means accepted, not
  applied. An omitted key means "leave unchanged"; a `null` value means **delete**.
- **No idempotency contract**, **no pagination**, **no rate-limit signalling**, and no
  request-id header.
- **Proprietary error envelope** `{errorCode, displayError, detailedError}` — not RFC 9457.
- **Protobuf timestamps** `{seconds, nanos}` rather than RFC 3339 strings, leaking the
  gRPC control plane through the REST surface.
- The bundle declares **no operationIds, no tags and no operation summaries**, so the
  generated SDK method names (`v1_edges_summary_get`) act as the de-facto operation
  identifiers.

## Artifacts

| Directory | Contents |
|---|---|
| `openapi/` | the verbatim Graphiant OpenAPI 3.0.0 bundle, v26.7.0 |
| `overlays/` | API Evangelist enhancements + recorded contract gaps (never mutates the original) |
| `authentication/` | token model, SSO/MFA, MSP tenant switching, permission domains |
| `conventions/` | versioning, async writes, declarative config semantics, naming, timestamps |
| `errors/` | error envelopes and the declared 4xx/5xx catalog |
| `lifecycle/` | release trains, deprecation posture, SLA and severity matrix, status page |
| `changelog/` | 20 dated releases derived from the SDK changelog |
| `data-model/` | 25 entities and 28 relationships derived from the spec |
| `conformance/` | standards conformance plus network standards and the compliance program |
| `packages/` | Python SDK, Go SDK, Ansible collection |
| `cli/` | the `graphiant` CLI command surface |
| `asyncapi/` | outbound webhook and notification-integration catalog (no AsyncAPI published) |
| `mcp/` | candidate MCP tool set and the REST tool crosswalk |
| `skills/` | five packaged agent skills, every operation verified against the spec |
| `security/` | domain security probe and trust center |
| `well-known/` | `/.well-known/` probe results (none published) |
| `llms/` | Graphiant's own `llms.txt` from the docs and marketing hosts |

No A2A agent card and no `/.well-known/security.txt` were found on any Graphiant host.
Graphiant publishes no hosted MCP server; its agentic surface, Gina AI, is an in-portal
assistant reached through authenticated `/v2/assistant/*` REST operations.
