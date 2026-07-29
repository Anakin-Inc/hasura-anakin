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

The full 21-capability Anakin surface — scraping, search, Wire automation,
website monitoring, AI visibility, sessions, and browser automation:

| Operation | HTTP | NDC kind | Notes |
|---|---|---|---|
| `submitUrlScraper` | `POST /url-scraper` | procedure (mutation) | async — returns `jobId` |
| `getUrlScraperJob` | `GET /url-scraper/{jobId}` | function (query) | poll until `status` is `completed`/`failed` |
| `search` | `POST /search` | procedure (mutation)* | synchronous, no polling |
| `submitAgenticSearch` | `POST /agentic-search` | procedure (mutation) | async — returns `jobId` |
| `getAgenticSearchJob` | `GET /agentic-search/{jobId}` | function (query) | poll until `status` is `completed`/`failed` |
| `submitMap` | `POST /map` | procedure (mutation) | async — returns `jobId` |
| `getMapJob` | `GET /map/{jobId}` | function (query) | poll until `status` is `completed`/`failed` |
| `submitCrawl` | `POST /crawl` | procedure (mutation) | async — returns `jobId` |
| `getCrawlJob` | `GET /crawl/{jobId}` | function (query) | poll until `status` is `completed`/`failed` |
| `wireResolve` | `GET /wire/resolve` | function (query) | synchronous — ranked candidate actions from an intent |
| `listWireCatalog` | `GET /wire/catalog` | function (query) | every catalog + action count |
| `getWireCatalog` | `GET /wire/catalog/{slug}` | function (query) | one catalog's full action list + param schemas |
| `submitWireTask` | `POST /wire/task` | procedure (mutation) | runs both read *and* write Wire actions — sync (data inline) or async (`job_id`) depending on the action |
| `getWireJob` | `GET /wire/jobs/{jobId}` | function (query) | poll until `status` is `completed`/`failed` |
| `listWireIdentities` | `GET /wire/identities` | function (query) | saved identities + credentials |
| `wireLogin` | `POST /wire/login` | procedure (mutation) | credentials-mode sign-in → `credential_id` |
| `wireBuildRequest` | `POST /wire/build-request` | procedure (mutation) | requests a new action for an uncataloged site; async, starts `pending` |
| `createMonitor` | `POST /monitors` | procedure (mutation) | recurring, billed, side-effecting |
| `listMonitors` | `GET /monitors` | function (query) | |
| `getMonitor` | `GET /monitors/{id}` | function (query) | |
| `deleteMonitor` | `DELETE /monitors/{id}` | procedure (mutation) | irreversible |
| `getMonitorChanges` | `GET /monitors/{id}/changes` | function (query) | |
| `pauseMonitor` / `resumeMonitor` / `runMonitorNow` | `POST /monitors/{id}/{pause,resume,run}` | procedure (mutation) | |
| `submitAiVisibilitySearch` | `POST /ai-visibility/search` | procedure (mutation) | async — poll by `search_id` |
| `getAiVisibilitySearch` | `GET /ai-visibility/search/{searchId}` | function (query) | poll until `status` is not `running` |
| `listAiVisibilitySources` | `GET /ai-visibility/sources` | function (query) | |
| `listSessions` | `GET /sessions` | function (query) | |
| `deleteSession` | `DELETE /sessions/{id}` | procedure (mutation) | irreversible |
| `submitBrowserTask` | `POST /ai/evaluate` | procedure (mutation) | always submitted async (`async: true`) — returns `workflow_id` |
| `getBrowserTaskJob` | `GET /ai/jobs/{id}` | function (query) | poll until `status` is `completed`/`failed`/`timed_out` |

\* `ndc-http` maps every non-`GET` method to an NDC procedure regardless of
whether the underlying call has side effects — `search`, `wireResolve`-style
GETs aside, several of the above (e.g. `search`, `submitWireTask` for a
read-type Wire action) are read-only but still land as mutation fields in
the generated GraphQL schema, same as `getAccountsAccount`-style POST
lookups in `ndc-stripe`. This is a property of the generic connector, not
something specific to Anakin.

The `formats` / `poll_timeout` SDK conveniences (client-side response
shaping, not distinct HTTP endpoints) remain out of scope — there is no
separate wire format for them to expose.

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
