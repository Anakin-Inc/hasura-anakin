# ndc-anakin (connector implementation)

Would-be contents of a **new, separate repository** (e.g. `github.com/anakin-io/ndc-anakin`,
following the real `hasura/ndc-stripe` and `hasura/ndc-sendgrid` layout) — the
thing that actually gets built into a Docker image and released as a
`connector-definition.tgz`, which `../registry-entry/releases/v0.1.0/connector-packaging.json`
then points at. It is **not** part of the `hasura/ndc-hub` repo itself; see
`../SUBMIT.md` for how the two pieces relate.

## What this is

A [Native Data Connector](https://hasura.github.io/ndc-spec/) for [Anakin](https://anakin.io)
(web scraping, AI search, and agentic research) built on Hasura's
[`ndc-http`](https://github.com/hasura/ndc-http) — a config-driven, no-code
HTTP-to-NDC engine. There is no custom Rust/TypeScript connector code: the
whole connector is an OpenAPI document (`config/anakin.openapi.json`) plus a
runtime config file (`config/config.yaml`), layered onto the generic
`ndc-http` Docker image (`Dockerfile`). This is the same pattern Hasura uses
for its own Stripe and SendGrid connectors.

## Covers

The three endpoints specified as this integration's baseline (matching what
every other single-purpose Anakin integration built this session covers):

| Operation | HTTP | NDC kind | Notes |
|---|---|---|---|
| `submitUrlScraper` | `POST /url-scraper` | procedure (mutation) | async — returns `jobId` |
| `getUrlScraperJob` | `GET /url-scraper/{jobId}` | function (query) | poll until `status` is `completed`/`failed` |
| `search` | `POST /search` | procedure (mutation)* | synchronous, no polling |
| `submitAgenticSearch` | `POST /agentic-search` | procedure (mutation) | async — returns `jobId` |
| `getAgenticSearchJob` | `GET /agentic-search/{jobId}` | function (query) | poll until `status` is `completed`/`failed` |

\* `ndc-http` maps every non-`GET` method to an NDC procedure regardless of
whether the underlying call has side effects — `search` is read-only but
still a mutation field in the generated GraphQL schema, same as
`getAccountsAccount`-style POST lookups in `ndc-stripe`. This is a property
of the generic connector, not something specific to Anakin.

Map/crawl/wire/sessions and the `formats`/`poll_timeout` SDK conveniences are
out of scope for this first pass — same scoping call made in the sibling
Fivetran and Dagster submissions in this directory.

## Async jobs are not polled by the connector

`ndc-http` (and NDC generally) has no concept of "submit, then wait." Submit
and poll are exposed as two independent GraphQL fields, exactly mirroring the
raw REST API. A Hasura DDN caller (or PromptQL) issues the submit mutation,
then re-issues the poll query until it observes a terminal `status` —
identical to how the Python SDK's own `client.scrape()` polls under the hood
(`anakin-py/src/anakin/client.py`), just moved up a layer to the GraphQL
consumer.

## Local dev

```sh
cp .env.example .env      # fill in ANAKIN_API_KEY, or point ANAKIN_SERVER_URL
                            # at the WireMock stack in ../registry-entry/tests/
docker-compose up --build
```

Serves the NDC HTTP service on `http://localhost:8080` — `/schema` returns
the generated NDC schema, `/query` and `/mutation` are the DDN engine's
proxy targets. Not run in this sandbox (no Docker); see `../SUBMIT.md`.

## Optional: pre-compile the schema

`config/config.yaml` points `ndc-http` at `anakin.openapi.json` directly and
lets it transform OpenAPI → NDC schema at container start (a documented,
supported mode). `ndc-stripe` instead pre-compiles with the `ndc-http-schema`
CLI into a static `schema.json` for faster cold starts:

```sh
go install github.com/hasura/ndc-http/ndc-http-schema@latest
ndc-http-schema convert -c generator/config.yaml -o config/schema.json
```

Not done here — the CLI needs a Go toolchain, unavailable in this sandbox.
Direct OpenAPI reference is fully functional without it; pre-compiling is a
later optimization, not a correctness requirement.
