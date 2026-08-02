# Unqork

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
