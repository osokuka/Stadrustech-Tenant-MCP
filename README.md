# Stadrustech-Tenant-MCP

Tenant-staff package for **Scardus Tech** Hosted Tenant MCP — connect your AI agent to the ITSM tenant console (tickets, assignment, clients, projects, invoices, reports).

| | |
|--|--|
| **Console** | `https://ticket.scardustech.com` (sign in as a **tenant staff** user) |
| **MCP URL** | `https://ticket.scardustech.com/tenant-mcp` |
| **Auth** | Tenant API key `sk_live_…` (`X-API-Key` or `Authorization: Bearer`), **or** OAuth Sign-in from Claude / ChatGPT / Gemini / Copilot |
| **Not accepted** | Client keys `ck_live_…` or Client MCP URL `/mcp` |

This repo is what **tenant staff / integrators download**. The MCP server is hosted by Scardus Tech; you do not run Docker yourself.

**Private distribution** — do not publish keys or clone into public repos with secrets.

## What you get

- **Cursor skill** `itsm-tenant-mcp-agent` — discovers clients by name, opens/assigns/status tickets, plans projects, runs reports — confirms before writes
- **MCP config snippets** for Cursor / Claude Desktop / ChatGPT / Gemini / Copilot
- **OAuth host guide** (`docs/OAUTH_CLIENTS.md`) — connect without pasting a key
- **Tool reference** for agents (`docs/TOOLS.md`)

## 1. Where to get your API key (`sk_live_…`)

Keys are created in the **tenant console** (Setup → API Keys) by a tenant admin.

1. Open **[https://ticket.scardustech.com](https://ticket.scardustech.com)**
2. Sign in with a **tenant staff** account — not a client portal login.
3. Open your profile / Setup → **API Keys**.
4. Create a key, grant the `can_*` flags you need (tickets, assign, projects, invoices, …).
5. **Copy the secret immediately** — shown **once**. It must start with `sk_live_`.
6. Store it in a password manager. **Never** commit it to git.

The API Keys screen also shows a **Hosted Tenant MCP** config snippet you can copy.

## 2. Connect Cursor (remote MCP)

Merge into your Cursor MCP settings (or use `configs/cursor-mcp.json.example`):

```json
{
  "mcpServers": {
    "itsm-tenant": {
      "url": "https://ticket.scardustech.com/tenant-mcp",
      "headers": {
        "X-API-Key": "sk_live_YOUR_KEY_HERE"
      }
    }
  }
}
```

Restart Cursor / reload MCP after saving.

## 3. Install the agent skill

**Option A — use this repo as a Cursor project**

```bash
git clone git@github.com:osokuka/Stadrustech-Tenant-MCP.git
cd Stadrustech-Tenant-MCP
```

Open the folder in Cursor. The skill under `.cursor/skills/itsm-tenant-mcp-agent/` is picked up automatically.

**Option B — copy into any project**

```bash
cp -R .cursor/skills/itsm-tenant-mcp-agent /path/to/your-project/.cursor/skills/
```

Or into your user skills: `~/.cursor/skills/itsm-tenant-mcp-agent/`.

## 4. Use it — staff workflows

Describe what you need in plain language. The agent should:

1. Resolve clients / tickets by **name** (`search` / `clients_resolve`) before UUIDs
2. Confirm before create / assign / bulk / status changes
3. Respect SLA rules (`pending` needs a comment; prefer `resolved` then `closed`)
4. Use technician **`user_id`** (Django PK) for assignment — not TenantUser UUID

Examples:

- “Open a high-priority ticket for Caritas — VPN down at Prishtina site, assign best tech.”
- “Monthly SLA pack for client X for last month.”
- “Create a LAN expansion project with switch install + cabling activities and export the offer PDF.”

## Security

- Never share `sk_live_` keys in chat logs or public repos.
- Keys are scoped to **one tenant** (all clients under that tenant). Least-privilege `can_*` flags.
- Unauthenticated MCP sessions are refused (401). Invalid keys are rejected (403). `ck_live_` is refused.
- Do **not** point this agent at Client MCP (`/mcp`).

## OAuth (Claude, ChatGPT, Gemini, Copilot)

Prefer **Sign in** in the host’s connector UI with MCP URL `https://ticket.scardustech.com/tenant-mcp`. Full steps: [docs/OAUTH_CLIENTS.md](docs/OAUTH_CLIENTS.md). Sign in as **tenant staff**; revoke under tenant **Profile → Connections**.

## Docs

- [Agent skill](.cursor/skills/itsm-tenant-mcp-agent/SKILL.md)
- [Field & workflow reference](.cursor/skills/itsm-tenant-mcp-agent/reference.md)
- [MCP tools](docs/TOOLS.md)
- [OAuth clients](docs/OAUTH_CLIENTS.md)
- Config examples under [`configs/`](configs/)

## Related

- Client (portal) package: [Stadrustech-MCP](https://github.com/osokuka/Stadrustech-MCP) — `ck_live_` / `/mcp` only

## Support

Contact your Scardus Tech platform admin, or open / assign tickets via Tenant MCP once connected.
