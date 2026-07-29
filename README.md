# hasura-anakin

Native Data Connector for [Anakin](https://anakin.io) — web scraping, AI
search, and multi-stage agentic research — for Hasura's GraphQL Engine v3
(Hasura DDN), submitted to the [ndc-hub](https://github.com/hasura/ndc-hub)
connector registry.

## Approach

Built on [`ndc-http`](https://github.com/hasura/ndc-http), Hasura's
config-driven "HTTP-to-NDC" connector — not a custom Rust/TypeScript
connector. The whole implementation is an OpenAPI 3.0 document plus a
runtime config file, layered onto the generic `ndc-http` Docker image. This
is the real, current, lower-lift path for wrapping a REST API: it's also how
Hasura's own [`ndc-stripe`](https://github.com/hasura/ndc-stripe) and
[`ndc-sendgrid`](https://github.com/hasura/ndc-sendgrid) connectors are
built. A from-scratch connector using the
[Rust SDK](https://github.com/hasura/ndc-sdk-rs) or
[reference implementation](https://github.com/hasura/ndc-spec/tree/main/ndc-reference)
would have been the much bigger, unwarranted lift for a REST API this shape.

## Two pieces, because ndc-hub itself is a registry, not a connector host

`hasura/ndc-hub` doesn't contain connector source code. It contains a
*registry entry per connector* — metadata, README, and a
`connector-packaging.json` per release pointing at a `.tgz` built and hosted
**elsewhere** (verified by reading `registry/hasura/stripe/` and
`registry/hasura/http/` directly). So there are two deliverables here:

```
connector/         The actual connector — OpenAPI spec + ndc-http config +
                    Dockerfile. Would live in its own repo
                    (e.g. github.com/anakin-io/ndc-anakin), get built into a
                    Docker image, and released as a connector-definition.tgz.

registry-entry/    The registry listing — metadata.json, README.md,
                    releases/v0.1.0/connector-packaging.json, and e2e
                    tests. This is what actually gets PR'd into
                    hasura/ndc-hub, at registry/anakin/anakin/.
```

See `SUBMIT.md` for exactly what's verified, what isn't, and the real steps
to file the PR — including a finding worth reading before investing further:
every connector currently in the ndc-hub registry, including recently added
ones, is Hasura-owned. There is no observed precedent of a third-party
namespace being merged.

## Covers

The full 21-capability Anakin surface — all 30 paths / 32 operations across
scraping, search, Wire automation, monitoring, AI visibility, sessions, and
browser automation:

- `POST /url-scraper` (submit) + `GET /url-scraper/:jobId` (poll)
- `POST /search` — synchronous, no polling
- `POST /agentic-search` (submit) + `GET /agentic-search/:jobId` (poll)
- `POST /map` (submit) + `GET /map/:jobId` (poll)
- `POST /crawl` (submit) + `GET /crawl/:jobId` (poll)
- `GET /wire/resolve` — synchronous action discovery from a natural-language intent
- `GET /wire/catalog` and `GET /wire/catalog/:slug` — browse the Wire catalog
- `POST /wire/task` (submit — sync or async) + `GET /wire/jobs/:jobId` (poll) —
  the single endpoint backing both read and write Wire actions
- `GET /wire/identities` — saved identities/credentials
- `POST /wire/login` — credentials-mode sign-in
- `POST /wire/build-request` — request a new Wire action for an uncataloged site
- `POST /monitors` (create), `GET /monitors` (list), `GET /monitors/:id` (get),
  `DELETE /monitors/:id` (delete), `GET /monitors/:id/changes`,
  `POST /monitors/:id/{pause,resume,run}`
- `POST /ai-visibility/search` (submit) + `GET /ai-visibility/search/:searchId` (poll)
- `GET /ai-visibility/sources`
- `GET /sessions` + `DELETE /sessions/:id`
- `POST /ai/evaluate` (submit, async) + `GET /ai/jobs/:id` (poll)

Ground truth for the original three endpoints (`url-scraper`, `search`,
`agentic-search`) was read directly from `anakin-py/src/anakin/client.py` and
`anakin-py/src/anakin/models.py`. Ground truth for the 18 endpoints added
afterward was read directly from `anakin-mcp/src/client.ts` and
`anakin-mcp/src/tools/*.ts` — the TypeScript MCP server's own HTTP client and
per-tool parameter schemas, which mirror the same production API
(`https://api.anakin.io/v1`, `X-API-Key` header auth). Several response
shapes there are only as concrete as the ground truth itself: `client.ts`
types a number of Wire/monitor/session/AI-visibility responses as `unknown`
or `Record<string, unknown>`, so the corresponding OpenAPI schemas
(`WireCatalogResponse`, `MonitorResponse`, `SessionsListResponse`, etc.) are
deliberately permissive (`additionalProperties: true`, no fabricated
required fields) rather than guessing field names the ground truth doesn't
confirm.
