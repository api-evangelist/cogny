# auth.md

You are an agent. This service supports **agentic registration**: discover → register → call API → handle revocation. Follow the steps in order; do not skip ahead.

This document conforms to the [auth.md protocol](https://github.com/workos/auth.md). Today Cogny supports the **anonymous** registration mode only. The OTP claim ceremony and ID-JAG identity assertions described in the upstream spec are planned (see [Notes for future spec parity](#notes-for-future-spec-parity)).

Real hosts:

- `https://app.cogny.com/mcp` — the MCP resource server (the API you want to call)
- `https://app.cogny.com` — the authorization server (registration lives here)

## Step 1 — Discover

Discovery is two hops — you may have already done this.

The 401 response that pointed you here also carries a `WWW-Authenticate` header with the PRM URL:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer resource_metadata="https://app.cogny.com/.well-known/oauth-protected-resource"
```

Pull the `resource_metadata` value from that header and fetch it (1a). If you don't have the 401 in hand, the conventional path on the resource server is `/.well-known/oauth-protected-resource`.

### 1a. Fetch the Protected Resource Metadata

```http
GET /.well-known/oauth-protected-resource
```

Response shape:

```json
{
  "resource": "https://app.cogny.com/mcp",
  "resource_name": "Cogny",
  "resource_logo_uri": "https://app.cogny.com/cogny-mark.svg",
  "authorization_servers": ["https://app.cogny.com"],
  "scopes_supported": ["read", "write", "tickets:read", "tickets:write"],
  "bearer_methods_supported": ["header"]
}
```

### 1b. Fetch the Authorization Server metadata

```http
GET https://app.cogny.com/.well-known/oauth-authorization-server
```

The `agent_auth` block in the response is the part written for you:

```json
{
  "agent_auth": {
    "skill": "https://app.cogny.com/auth.md",
    "register_uri": "https://app.cogny.com/api/agent/auth",
    "identity_types_supported": ["anonymous"],
    "anonymous": {
      "credential_types_supported": ["api_key"]
    }
  }
}
```

The outer metadata fields (`issuer`, `token_endpoint`, etc.) describe the standard OAuth Authorization Code flow used by interactive clients (Claude Cowork, ChatGPT). Agents register via `agent_auth.register_uri` instead — no browser bounce.

## Step 2 — Pick a method

Only one method is wired up today:

- **anonymous** → [Step 3](#step-3--register).

You can detect availability programmatically from `identity_types_supported`. The spec's `identity_assertion` modes (verified-email + OTP, and ID-JAG) will appear there as we ship them.

## Step 3 — Register

```http
POST https://app.cogny.com/api/agent/auth
Content-Type: application/json

{
  "type": "anonymous",
  "requested_credential_type": "api_key",
  "email": "user@example.com",
  "agent": "claude-code"
}
```

Field notes:

- `type` *(required)* — `"anonymous"`.
- `requested_credential_type` *(required)* — `"api_key"`. This is the only credential shape Cogny mints today.
- `email` *(optional but recommended; Cogny extension to the anonymous flow)* — if present, the workspace is provisioned under this email so the user can later sign in to the web UI without losing data. If absent, the workspace is bound only to the `registration_id` and the user must claim it before they can recover access from a browser.
- `agent` *(optional)* — your runtime slug (`claude-code`, `codex`, `cursor`, `gemini-cli`, …). Analytics only; not validated.

Before sending, surface the service's `resource_name` (`Cogny`) and `resource_logo_uri` to the user along with the scope set (`["read", "write"]`), and confirm. This is the user's only consent gate today.

Response (200):

```json
{
  "registration_id": "8c1f2a4e-5b3d-4c91-a0e7-6f9b8d2a4c1e",
  "registration_type": "anonymous",
  "credential_type": "api_key",
  "credential": "cogny_lite_aBcD1234...",
  "credential_expires": null,
  "scopes": ["read", "write"]
}
```

What each field tells you:

- `registration_id` — the Cogny workspace UUID. Persist this; you'll need it to refer to the workspace later.
- `credential_type: "api_key"` — present this as a `Bearer` token (see Step 5).
- `credential_expires: null` — API keys don't time-expire, but they can be revoked by the workspace owner from the Cogny web UI.
- `scopes` — same as `post_claim_scopes` today (pre-claim and post-claim are equal). When the claim ceremony ships, the immediate credential will drop to read-only and post-claim will unlock write.

Cogny does not return `claim_url` / `claim_token` today — the claim ceremony is not yet implemented. Go to [Step 5](#step-5--use-the-credential).

## Step 4 — Claim ceremony (not yet available)

The auth.md protocol describes an OTP-based claim ceremony for upgrading an anonymous registration to a verified identity. Cogny will implement this before charging anyone (the 15-credit signup gift is not gated, but the Stripe checkout step will be). Until then, the registration is final at Step 3 and write scope is granted immediately.

## Step 5 — Use the credential

Present the credential as a Bearer token against the MCP endpoint:

```http
POST https://app.cogny.com/mcp
Authorization: Bearer cogny_lite_aBcD1234...
Content-Type: application/json

{ "jsonrpc": "2.0", "method": "initialize", "id": 1,
  "params": { "protocolVersion": "2025-06-18",
              "capabilities": {},
              "clientInfo": { "name": "your-agent", "version": "0.0.0" } } }
```

The MCP endpoint speaks JSON-RPC over Streamable HTTP. Standard flow:

1. `initialize` → server handshake
2. `notifications/initialized` (no response)
3. `tools/list` → see what you can call
4. `tools/call` for each tool

Free-tier workspaces start with the `cogny_status` tool (and a small set of zero-cost reads); the full ~250-tool surface (BigQuery, Google Ads, Meta Ads, GA4, Search Console, TikTok, X, LinkedIn, Mailchimp, HubSpot, Shopify, …) unlocks after the workspace subscribes to Solo ($9/mo) or Cloud ($499/mo). Call `cogny_status` first to see what's available in this workspace.

The same Bearer also authenticates non-MCP Cogny endpoints scoped to the workspace (`/api/lite/*`).

### Terminal-agent shortcut (optional)

If you're a terminal-based agent (Claude Code, Codex, Gemini CLI, etc.), the `@cogny/cli` npm package wraps Step 3 + Step 5 into shell commands so you don't have to hand-roll the JSON-RPC:

```bash
# `npx` avoids the global-install PATH friction that bites
# nvm / Homebrew node / fresh Debian setups. No shell-config
# edits needed.
npx @cogny/cli init --agent claude-code
npx @cogny/cli tools list
npx @cogny/cli tools call <name> --input '<json>'
```

`npx @cogny/cli init` POSTs to `/api/agent/auth` under the hood, persists the credential to `~/.cogny/config.json`, and the subsequent `tools call` invocations send the Bearer for you. Pure-spec agents that want to call `/mcp` directly should keep using the JSON-RPC flow above — the CLI is just sugar for the common case.

If you get a 401 on a previously-working credential: drop it, restart at [Step 1](#step-1--discover). Do not stash the credential and retry.

Full MCP tool reference: `https://docs.cogny.com/mcp`.

## Errors

| Code                          | Where               | What to do                                                                                          |
| ----------------------------- | ------------------- | --------------------------------------------------------------------------------------------------- |
| `invalid_email`               | `/api/agent/auth`   | Email shape failed validation. Re-check and re-send.                                                |
| `disposable_email`            | `/api/agent/auth`   | Email domain is on the disposable-domain blocklist. Use a real address.                             |
| `unsupported_credential_type` | `/api/agent/auth`   | We mint `api_key` only today. Re-send with `requested_credential_type: "api_key"`.                  |
| `anonymous_not_enabled`       | `/api/agent/auth`   | Not returned today; reserved for when we want to gate anonymous signups.                            |
| `rate_limited` (429)          | `/api/agent/auth`   | More than 5 signups from this IP in the last hour. Back off and retry.                              |

Retry policy:

- 5xx → exponential backoff, retry the same request.
- 4xx → do not retry the same payload; act on the table above.
- 401 on a previously-working credential → drop the credential and restart at [Step 1](#step-1--discover).

## Revocation

You do not initiate revocation yourself. The workspace owner revokes credentials from the Cogny web UI. On a 401 for a previously-working credential, drop it and restart at Step 1.

## Notes for future spec parity

- **Verified-email + OTP** (`identity_assertion` with `assertion_type: "verified_email"`): tracked behind the Stripe-checkout gate. Will be the first claim ceremony Cogny ships.
- **ID-JAG** (`identity_assertion` with `assertion_type: "urn:ietf:params:oauth:token-type:id-jag"`): we'll add JWKS verification once a major agent provider (Anthropic, OpenAI, Google) ships ID-JAG issuance. The `agent_auth` block will advertise availability the same day.
- **Provider-driven revocation** (`revocation_uri`): tied to ID-JAG support.
