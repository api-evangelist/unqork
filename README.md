# Unqork

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

Unqork is an enterprise application development platform — a no-code / "codeless" platform-as-a-service used by banks, insurers, healthcare organizations and government agencies to build and operate complex, regulated business applications without hand-written application code.

- Website: https://unqork.com/
- Developer portal: https://developers.unqork.io/
- Documentation: https://docs.unqork.io/
- Community: https://community.unqork.com/
- Status: https://status.unqork.com/

## API

**Unqork Customer API** — OpenAPI 3.0.3, 93 operations across 64 paths, harvested verbatim from
`https://developers.unqork.io/api/1.0/openapi.yml` (the ReDoc `spec-url` behind the developer portal).

- Base URL: `https://{subdomain}.unqork.io/api/1.0` — per-tenant, per-environment
- Auth: OAuth 2.0 bearer token (client credentials + password grant), one-hour tokens
- Authorization: RBAC via Express/Creator roles — **no OAuth scopes**
- Errors: custom `{code, message}` JSON, **not** RFC 9457; `412` carries the validation/execution envelope
- Paging: offset/limit
- Rate limits: per source IP, per server, per 60s — default 1000, minimum 100, configurable per environment;
  signalled with `x-ratelimit-limit` / `-remaining` / `-reset`
- **No idempotency contract** — no `Idempotency-Key`, no `If-Match`, no `ETag`

## Artifacts

| Directory | What's in it |
|---|---|
| `openapi/` | The harvested Unqork Customer API specification |
| `authentication/` · `scopes/` | OAuth 2.0 profile and why there are no scopes |
| `conventions/` | Cross-cutting request/response semantics |
| `errors/` | Error catalogue derived from the spec + live probes |
| `rate-limits/` | Documented limits and observed headers |
| `lifecycle/` · `changelog/` | Release cadence, versioning, credential lifecycle |
| `data-model/` | Entity-relationship graph derived from the spec |
| `conformance/` | Standards conformance and published compliance programs |
| `security/` | Domain-security probe, vulnerability disclosure, trust center |
| `well-known/` | `/.well-known/` probe record (all misses) |
| `packages/` · `components/` | Package registry findings and the BYO Component SDK |
| `sandbox/` | Environment-based test/live separation and the Training environment |
| `asyncapi/` | Inbound webhook receiver surface (no AsyncAPI published) |
| `mcp/` | Candidate agent tool surface derived from the spec (Unqork ships no MCP server) |
| `agentic-access/` | Per-operation `x-agentic-access` execution contracts |
| `skills/` | Five packaged Agent Skills grounded in real operationIds |
| `overlays/` | OpenAPI Overlay 1.0.0 of API Evangelist enhancements |
| `llms/` | Unqork's own `llms.txt`, saved verbatim |

Profiled by the [API Evangelist](https://apievangelist.com) enrichment pipeline. See `apis.yml` for the
full APIs.json index.
