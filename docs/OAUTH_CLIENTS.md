# Connect Hosted Tenant MCP via OAuth (Claude, ChatGPT, Gemini, Copilot)

You can attach Scardus **Hosted Tenant MCP** to major AI hosts without pasting an API key. The host opens a browser; you sign in with a **tenant staff** account. Connections are listed under **My Profile → Connections** in the tenant console and can be revoked anytime.

| | |
|--|--|
| **MCP URL** | `https://ticket.scardustech.com/tenant-mcp` |
| **Console** | `https://ticket.scardustech.com` |
| **Who can sign in** | Tenant staff users only (not client portal accounts) |

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

**Hosted / Enterprise custom MCP:** register `https://ticket.scardustech.com/tenant-mcp` in the Gemini admin UI and complete Google’s OAuth redirect.

**Gemini CLI (loopback):** configure remote MCP with this URL + OAuth (or `sk_live_` header).

Key-based example: [`configs/gemini_mcp.example.json`](../configs/gemini_mcp.example.json).

---

## Microsoft 365 Copilot / Teams

1. Agents Toolkit or Copilot Studio → MCP endpoint `https://ticket.scardustech.com/tenant-mcp`.
2. Complete OAuth; sign in as tenant staff.
3. Revoke later under console **Profile → Connections**.

Key-based example: [`configs/copilot_mcp.example.json`](../configs/copilot_mcp.example.json).

---

## Cursor / local CLIs

Prefer header API key (`configs/cursor-mcp.json.example`). OAuth loopback is also supported by the authorization server when the host initiates Sign-in against `/tenant-mcp`.

---

## Revoke

Tenant console → **My Profile → Connections** → **Revoke**. The access key is deactivated immediately (MCP auth cache is short).
