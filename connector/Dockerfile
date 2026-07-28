# Thin layer on the generic, config-driven ndc-http connector — no custom
# code. Mirrors the real hasura/ndc-stripe and hasura/ndc-sendgrid Dockerfiles
# (both also "HTTP Connector + this vendor's OpenAPI spec").
#
# Pin the tag to the ndc-http release verified against this config before
# publishing (v0.14.1 was the latest release read from the ndc-hub registry
# at research time — confirm against https://github.com/hasura/ndc-http/releases
# before building for real).
FROM ghcr.io/hasura/ndc-http:v0.14.1

COPY ./config /etc/connector
