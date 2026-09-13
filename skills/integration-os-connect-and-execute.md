---
name: one-connect-and-execute
description: >-
  Connect an end user's third-party account to One and execute an action on it through the
  Passthrough API. Use when an agent needs to read or write on Gmail, Slack, Stripe, HubSpot,
  Shopify or any of the 789 platforms One fronts, and the connection does not exist yet.
api: One API (https://api.withone.ai)
provider: IntegrationOS
providerId: integration-os
generated: '2026-09-13'
method: generated
source: >-
  operationIds verified against openapi/integration-os-one-api-openapi.json;
  conventions from conventions/integration-os-conventions.yml and
  https://www.withone.ai/docs/api-reference/introduction
operations:
  - list_connectors
  - generate_authkit_token
  - init_authkit
  - list_connections
  - search_actions
  - list_actions
  - passthrough
  - delete_connection
---

# Connect an account, then execute an action

One is a proxy, not a system of record. Nothing happens until a **connection** exists — an
authenticated link between one end user and one third-party platform, in one environment.

## Before you start

- Authenticate with the `x-one-secret` header. Keys are **environment-bound**: a Sandbox key cannot
  touch a Production connection, and connectors cannot be moved between environments once created.
- Decide the tenancy level first. Most operations have three siblings — unscoped,
  `/organizations/{org_id}`, and `/organizations/{org_id}/projects/{project_id}`. Pick one and stay
  there; a call at the wrong level returns 404, not 403.

## 1. Find out what can be connected

    GET /v1/available-connectors          -> list_connectors
    GET /v1/available-connectors/connected -> list_connected_connectors

`list_connectors` returns the catalogue. `list_connected_connectors` returns only what this account
has already wired up — check it first and skip the auth flow if the connection exists.

## 2. Put the user through the connect flow

    POST /v1/authkit/token   -> generate_authkit_token
    POST /v1/authkit         -> init_authkit

Mint the token on YOUR backend, hand it to the `@withone/auth` widget on the front end, and let the
user complete the grant. Never mint the token in the browser — it is a bearer credential derived
from your secret key.

## 3. Confirm the connection landed

    GET /v1/connections            -> list_connections
    GET /v1/connections/reachable  -> list_reachable_connections

`list_reachable_connections` is the one to trust: a connection can exist and still be unreachable
because its OAuth token failed to refresh. Note the `connectionKey` — it looks like
`live::gmail::default` and it is the handle every later call uses.

## 4. Find the action, then read it, then run it

    GET  /v1/available-actions/search/{platform} -> search_actions
    GET  /v1/available-actions/{platform}        -> list_actions
    POST /v1/passthrough/{key}                   -> passthrough

Search ranks by relevance, not by safety — the provider's own integration-standards skill warns that
asking Gmail for "draft" surfaces the **send** action right beside the create action. Read the title
and the path before you pick an `actionId`.

Execute through Passthrough, naming the connection and the action in headers:

    X-One-Connection-Key: live::gmail::default
    X-One-Action-Id: <actionId>

## Rules that will bite you

- **Passthrough is not reversible.** One documents no compensating action for a passthrough call.
  Whether a sent message, a created charge or a deleted record can be taken back is entirely the
  downstream platform's business, and One will not tell you. Treat every `passthrough` write as
  final unless you have separately confirmed the downstream platform's own undo path.
- **402 before 429.** Plan-quota exhaustion surfaces as `402 Payment Required` on 186 of 248
  operations; `429` appears on only 39. Handle both, and handle 402 first.
- **No rate-limit headers.** The published ceiling is 100/min (Free), 500/min (Starter),
  1,000/min (Pro) — but nothing on the wire tells you how close you are. Exponential backoff with
  jitter is the only safe retry.
- **No Idempotency-Key.** This API has no idempotency header. A retried `passthrough` POST is a
  second real call on the downstream platform. Do not blind-retry a write.
- **Quote the correlationId.** Every error body carries a required `correlationId`, echoed as the
  `x-one-correlation-id` response header. Capture it before you retry.

## Tearing down

    DELETE /v1/connections/{id} -> delete_connection

There is no restore. No undelete operation exists for any resource in this API.
