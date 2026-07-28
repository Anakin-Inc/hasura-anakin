# Anakin Connector

## Overview

The Anakin Connector provides an instant adapter for Hasura DDN to request
[Anakin](https://anakin.io) web scraping, AI search, and agentic research
resources via GraphQL. This connector is built upon the
[HTTP connector](https://github.com/hasura/ndc-http) and Anakin's
[OpenAPI specification](https://github.com/anakin-io/ndc-anakin/blob/main/config/anakin.openapi.json).

## Build on Hasura DDN

Get a free Anakin API key at [anakin.io/dashboard](https://anakin.io/dashboard)
(300 credits, no card required), then add the connector to your DDN project
and set `ANAKIN_API_KEY`.

## Fork the connector

You can fork the [connector's repo](https://github.com/anakin-io/ndc-anakin)
and iterate on it yourself.

## License

The Anakin connector is available under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).
