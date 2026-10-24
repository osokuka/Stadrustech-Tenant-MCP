# Connect Hosted Tenant MCP via OAuth (Claude, ChatGPT, Gemini, Copilot)

You can attach Scardus **Hosted Tenant MCP** to major AI hosts without pasting an API key. The host opens a browser; you sign in with a **tenant staff** account. Connections are listed under **My Profile → Connections** in the tenant console and can be revoked anytime.

| | |
|--|--|
| **MCP URL** | `https://ticket.scardustech.com/tenant-mcp` |
| **Console** | `https://ticket.scardustech.com` |
| **Who can sign in** | Tenant staff users only (not client portal accounts) |
| **OAuth AS** | `https://ticket.scardustech.com/api/tenant-mcp-oauth/` |

API keys (`sk_live_…`) still work for Cursor and any host that supports custom headers — see the root [README](../README.md).

**Do not** use Client MCP URL `/mcp` or `ck_live_` keys here.

---

## Claude

1. Claude → **Settings → Connectors** (or Claude Desktop custom MCP).
2. Add remote server: `https://ticket.scardustech.com/tenant-mcp`.
3. Choose **Sign in** / OAuth.
4. Log in with your **tenant staff** user and approve.

Key-based example: [`configs/claude_desktop_config.example.json`](../configs/claude_desktop_config.example.json).

---

## ChatGPT / OpenAI (Codex)

1. ChatGPT → **Settings → Apps & Connectors** → add a **custom MCP** connector.
2. Server URL: `https://ticket.scardustech.com/tenant-mcp`.
3. Complete the OpenAI OAuth flow.
4. Sign in as a **tenant staff** user.

Key-based example: [`configs/chatgpt_mcp.example.json`](../configs/chatgpt_mcp.example.json).

---

## Google Gemini

### Gemini Enterprise / Business (paste OAuth — no DCR)

Gemini Enterprise **does not support Dynamic Client Registration**. Choose **OAuth 2.0** (not “no authentication”) and paste:

| Field | Value |
|-------|--------|
| MCP Server URL | `https://ticket.scardustech.com/tenant-mcp` |
| Authorization URL | `https://ticket.scardustech.com/api/tenant-mcp-oauth/authorize` |
| Token URL | `https://ticket.scardustech.com/api/tenant-mcp-oauth/token` |
| Client ID | `gemini-tenant-mcp` |
| Client Secret | *(leave blank)* |
| Enable PKCE | **Yes** |
| Scopes | `mcp:tools offline_access` |

Sign in as a **tenant staff** user. Google’s redirect `https://vertexaisearch.cloud.google.com/oauth-redirect` is allowlisted on Scardus.

If the UI says *“authentication method that Gemini doesn’t support”*, you are on a discovery-only flow — use the Enterprise **Custom MCP → OAuth 2.0** form with the table above (not URL-only Connected Apps).

Ask your Scardus admin to run `ensure_mcp_oauth_gemini_clients` if Client ID `gemini-tenant-mcp` is unknown on the server.

### Connected Apps / Spark (URL discovery)

MCP URL `https://ticket.scardustech.com/tenant-mcp`. Discovery uses `401` + `WWW-Authenticate: Bearer resource_metadata=…` and public well-known under `/tenant-mcp/.well-known/` plus `/.well-known/oauth-protected-resource/tenant-mcp`.

### Gemini CLI (loopback)

Remote MCP with this URL + OAuth, or `sk_live_` header. Loopback callbacks are allowlisted.

Key-based example: [`configs/gemini_mcp.example.json`](../configs/gemini_mcp.example.json).

---

## Microsoft 365 Copilot / Teams

1. Agents Toolkit or Copilot Studio → MCP endpoint `https://ticket.scardustech.com/tenant-mcp`.
2. Complete OAuth; sign in as tenant staff.
3. Revoke later under console **Profile → Connections**.

Key-based example: [`configs/copilot_mcp.example.json`](../configs/copilot_mcp.example.json).

---

## Cursor / local CLIs

Prefer header API key (`configs/cursor-mcp.json.example`). OAuth loopback is also supported when the host initiates Sign-in against `/tenant-mcp`.

---

## Revoke

Tenant console → **My Profile → Connections** → **Revoke**. The access key is deactivated immediately (MCP auth cache is short).
