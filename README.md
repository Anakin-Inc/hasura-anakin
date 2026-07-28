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

Same three-endpoint baseline as every other single-purpose Anakin
integration built this session:

- `POST /url-scraper` (submit) + `GET /url-scraper/:jobId` (poll)
- `POST /search` — synchronous, no polling
- `POST /agentic-search` (submit) + `GET /agentic-search/:jobId` (poll)

Ground truth read directly from `anakin-py/src/anakin/client.py` and
`anakin-py/src/anakin/models.py` — base URL `https://api.anakin.io/v1`, auth
via `X-API-Key` header.
