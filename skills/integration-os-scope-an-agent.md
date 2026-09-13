---
name: one-scope-an-agent
description: >-
  Grant an AI agent the narrowest workable access to a One account and take it back afterwards. Use
  when wiring an agent to the hosted MCP server, minting a scoped API key, or auditing what an
  existing grant can actually do.
api: One API (https://api.withone.ai)
provider: IntegrationOS
providerId: integration-os
generated: '2026-09-13'
method: generated
source: >-
  operationIds verified against openapi/integration-os-one-api-openapi.json; OAuth metadata probed
  at https://mcp.withone.ai/.well-known/oauth-authorization-server and
  https://mcp.withone.ai/.well-known/oauth-protected-resource; consent model from
  https://www.withone.ai/docs/mcp
operations:
  - create_event_access
  - specify_connection_access
  - read_connection_access
  - update_connection_access
  - revoke_connection_access
  - list_oauth_authorizations
  - read_oauth_authorization
  - revoke_oauth_authorization
  - revoke_oauth_client_user
  - delete_event_access
---

# Scope an agent, then be able to take it back

One fronts 789 platforms and 111,176 actions. An unscoped grant is an agent that can do all of it.
There are two grant paths and they are enforced differently — know which one you are in.

## Path A — the hosted MCP server (OAuth consent)

The agent connects to `https://mcp.withone.ai/mcp` (Streamable HTTP) and signs in through OAuth.
The endpoint is a correctly-behaved RFC 9728 protected resource: an unauthenticated call returns
401 with

    WWW-Authenticate: Bearer resource_metadata="https://mcp.withone.ai/.well-known/oauth-protected-resource"

The consent screen is where scoping happens, and the choices are enforced **server-side on every
call**:

- organization or project, and sandbox or production;
- per-connection access level — read only, full, or an explicit set of actions;
- **knowledge-only mode**, which removes execution entirely. The agent can still list integrations,
  search actions and read documentation, but nothing runs against a live platform.

Reach for knowledge-only first. It is the closest thing One ships to a dry run, and most agent tasks
that look like they need execution only need discovery.

The grant is visible and revocable through the API:

    GET    /v1/oauth-authorizations             -> list_oauth_authorizations
    GET    /v1/oauth-authorizations/{client_id} -> read_oauth_authorization
    DELETE /v1/oauth-authorizations/{client_id} -> revoke_oauth_authorization
    DELETE /v1/oauth-clients/{client_id}/users/{user_id} -> revoke_oauth_client_user

Revoke the whole client, or just one user's grant on it. No window applies — revocation is immediate
and there is no undo.

## Path B — a scoped API key

    POST   /v1/event-access  -> create_event_access
    POST   /v1/access/{id}   -> specify_connection_access
    GET    /v1/access/{id}   -> read_connection_access
    PUT    /v1/access/{id}   -> update_connection_access
    DELETE /v1/access/{id}   -> revoke_connection_access

`specify_connection_access` attaches restrictions to an already-minted key: a global `methods` list
(e.g. `["GET"]`) plus per-connection `rules`. A rule targets a `connectionKey` such as
`live::gmail::default` and may narrow further to specific `actionIds`.

Two traps in that shape:

- **`revoke_connection_access` does not revoke the key.** It DELETEs the *restrictions*, which
  leaves the key with unrestricted access to every connection, platform and method. To actually kill
  a key, use `delete_event_access`. Read that sentence twice — the operation named "revoke" makes the
  key more powerful, not less.
- **An omitted `methods` list inherits or permits everything.** Set it explicitly. An
  `actionIds`-scoped rule never confers connection-record management, only the listed actions.

## Choosing scopes

The authorization server advertises 38 scopes in a `{level}:{resource}:{read|write}` grid, where
level is `user`, `org` or `project`:

    user:connections:read     org:connections:write     project:workflows:read
    user:secrets:write        org:authkit:read          project:ai_skills:write
    ...

The protected-resource document for the MCP endpoint advertises only two —
`user:connections:read` and `user:connections:write` — so an MCP agent needs far less than the full
grid. Ask for the two, not the thirty-eight.

Dynamic client registration is open at `https://mcp.withone.ai/oauth/register` with
`token_endpoint_auth_methods_supported: ["none"]` and PKCE `S256`. Note that the authorize and token
endpoints live on a **different host** (`api.withone.ai`) than the issuer (`mcp.withone.ai`); a
client that assumes one host will fail.

## Audit checklist

1. `list_oauth_authorizations` — who holds a live grant?
2. `read_connection_access` on every key — is any key unrestricted?
3. `list_reachable_connections` — what can those grants actually reach today?
4. Anything you cannot justify: `revoke_oauth_authorization` or `delete_event_access`.

## Rotation is one-way

`rotate_link`, `regenerate_oauth_client_secret` and `regenerate_ai_runner_dashboard_password` destroy
the previous secret. Keys are one-time-copyable in the dashboard (changelog v2.7.0), so a rotated
value that was not captured is gone. Rotate deliberately, and capture the new value in the same step.
