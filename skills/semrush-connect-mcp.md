---
name: semrush-connect-mcp
description: >-
  Connect an AI agent to the Semrush MCP server over streamable HTTP — complete the OAuth
  flow, understand what data the server actually fronts, and budget the API units each call
  consumes.
api: Semrush MCP
generated: '2026-08-13'
method: generated
source: https://developer.semrush.com/api/v4/introduction/semrush-mcp/
endpoint: https://mcp.semrush.com/v2/mcp
operations:
  - POST https://mcp.semrush.com/v2/mcp   # JSON-RPC: initialize, tools/list, tools/call
grounding: >-
  Endpoint, transport and auth model are read from the Semrush MCP documentation and
  confirmed by probe — POST to the endpoint returns 401 with an RFC 9728 WWW-Authenticate
  challenge. Tool names are deliberately not listed: tools/list is auth-gated and Semrush
  publishes no tool catalogue, so any list here would be invented.
---

# Connect an agent to Semrush MCP

## What you get

The Semrush MCP server fronts three of Semrush's APIs, and only three:

- the **Trends API** — limited to what your Trends subscription level entitles;
- the whole **SEO API**;
- the **read-only** methods of the **Projects API v3** — an agent can read your projects but
  cannot create them, change settings, or take any other action.

It does **not** front the Listing Management API, the Map Rank Tracker API, or the App
Center partner surface.

## Endpoint and transport

```
https://mcp.semrush.com/v2/mcp
```

**Streamable HTTP only.** There is no SSE endpoint and no stdio package — Semrush publishes
no `npx` or `pip` MCP server, so this URL is the entire distribution. A legacy `/v1/mcp` path
is still live.

## Authorization

### OAuth (default — do this)

No headers to configure. The agent registers dynamically with the Semrush OAuth server and
you are redirected to the Semrush login page to approve access.

The discovery chain, all anonymous and all verifiable before you commit:

1. `POST https://mcp.semrush.com/v2/mcp` → `401` with
   `WWW-Authenticate: Bearer resource_metadata="https://mcp.semrush.com/.well-known/oauth-protected-resource/v2/mcp"`
2. That document names the authorization server `https://oauth.semrush.com` and
   `scopes_supported: ["mcp.access"]`
3. `https://mcp.semrush.com/.well-known/oauth-authorization-server` gives the endpoints:
   - authorization `https://api.semrush.com/apis/v4/auth/v0/oauth2/auth`
   - token `https://api.semrush.com/apis/v4-raw/auth/v1/oauth2/access_token`
   - registration `https://api.semrush.com/apis/v4-raw/auth/v1/oauth2/register`
   - revocation `https://api.semrush.com/apis/v4/auth/v1/oauth2/revoke`

Grants: `authorization_code` and `refresh_token`. PKCE: `S256` and `plain` — **always
choose S256**; `plain` is advertised but is a downgrade.

`mcp.access` is a single coarse scope. Granting it grants everything your subscription
entitles; there is no way to narrow it to one API or to read-only.

### API key (fallback)

If your agent does not support OAuth, authenticate with a Semrush API key. Keys are
version-specific — use the version that matches the API you are reaching — and are issued
from the **API Keys** section of your Semrush profile. The full key value is shown once.

## Entitlement — check before you connect

**SEO data (Standard API).** One of: Semrush One Starter, Semrush One Pro+, SEO Classic Pro,
SEO Classic Guru — each includes 50,000 API units that refresh on your renewal date. Or SEO
Classic Business / Semrush One Advanced **plus** an API units package (2M, 5M, 10M or 20M).

**Traffic & market data (Trends API).** Any Trends API plan (Basic or Premium) includes MCP
access.

If neither applies, the connection will authenticate and then fail on entitlement.

## Budget — this is metered

MCP calls consume **the same API unit balance** as the REST APIs. Units are charged by report
type and by how much data comes back, and historical data costs several times live data.
An agent left to explore freely will burn a unit package fast.

Check the balance before and after a session — the call is free:

```
GET http://www.semrush.com/users/countapiunits.html?key=<key>
```

## Enumerating the tools

`tools/list` requires an authenticated session. Semrush publishes no tool catalogue in its
documentation and its `llms.txt` links only the MCP landing page, so the only way to learn
the tool names and input schemas is to complete OAuth and introspect the live server.
Do that once and record the result rather than assuming a shape.

## Documented clients

Claude, Claude Code, ChatGPT, Cursor, VS Code, Antigravity 2.0, Antigravity CLI, Perplexity,
Lovable. For anything else, Semrush asks you to contact Support — each request is reviewed
individually.

## Governance note

Semrush's Terms of Service (§3.3) forbid caching API-derived information for more than one
month without written consent. That applies to anything your agent persists from MCP
responses — memory files, vector stores and warehouse tables included.

## Related artifacts

- `mcp/semrush-mcp.yml` — the deployment record and probe evidence
- `mcp/semrush-tool-crosswalk.yml` — what the MCP surface does and does not map to
- `scopes/semrush-scopes.yml`
- `well-known/semrush-well-known.yml`
