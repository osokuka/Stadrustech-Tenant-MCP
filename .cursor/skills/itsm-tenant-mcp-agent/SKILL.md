---
name: itsm-tenant-mcp-agent
description: >-
  Drives Scardus Tech Hosted Tenant MCP (sk_live_) tools for tenant staff:
  search clients, per-client SLA service prices/catalog, tickets, projects,
  invoices, and reports. Never uses ck_live_ or Client MCP. Persistently
  gathers required fields and confirms before writes.
---

# ITSM Tenant MCP Agent (Stadrustech-Tenant-MCP)

Assist **tenant staff** through the **Hosted Tenant MCP** (`itsm-tenant`, `sk_live_…` only).

- Console (keys / OAuth connections): `https://ticket.scardustech.com` → **Setup → API Keys** or **Profile → Connections**
- MCP URL: `https://ticket.scardustech.com/tenant-mcp`
- OAuth hosts: Claude, ChatGPT, Gemini, Copilot — [docs/OAUTH_CLIENTS.md](../../../docs/OAUTH_CLIENTS.md)
- Tool list: [docs/TOOLS.md](../../../docs/TOOLS.md)
- Field & workflow detail: [reference.md](reference.md)

## If auth fails / no key

Tell the user clearly — either path works:

**OAuth (Claude / ChatGPT / Gemini / Copilot):** add MCP URL `https://ticket.scardustech.com/tenant-mcp` → **Sign in** with a **tenant staff** account → revoke later under **Profile → Connections**. **Gemini Enterprise:** paste OAuth fields from [docs/OAUTH_CLIENTS.md](../../../docs/OAUTH_CLIENTS.md) (Client ID `gemini-tenant-mcp`, PKCE, blank secret) — URL-only Connected Apps may reject auth.

**API key (Cursor and any header-capable host):**

1. Sign in at **https://ticket.scardustech.com** as a **tenant staff** user (not client portal).
2. Setup → **API Keys** → create a key with needed `can_*` flags → copy the `sk_live_…` secret (shown once).
3. Put it in MCP config as `X-API-Key` / Bearer (see repo README / `configs/`).
4. Copy the MCP snippet from the API Keys screen if available.

Never invent or guess a key. Never use `ck_live_` or Client MCP `/mcp`.

## Non-negotiables

1. **Auth:** `X-API-Key: sk_live_…` (or Bearer). Never `ck_live_`. Never Client MCP URL.
2. **Discover by name:** Call **`search`** or **`clients_resolve`** before inventing UUIDs.
3. **No guessing IDs:** Never fabricate ticket / client / project / invoice UUIDs.
4. **Confirm before write:** Summarize the planned tool call and get explicit OK before create / assign / bulk / status / delete-like updates.
5. **SLA:** `pending` via `tickets_set_status` **requires** a comment. Prefer `resolved` then `closed`. Assign sets `in_progress`.
6. **Assignment:** `technician_id` / `assigned_to` = Django **user PK** (`technicians[].user_id`), not TenantUser UUID.
7. **Service prices:** For quotes / rates call **`clients_service_prices(client_id)`** (or `clients_service_catalog`). Require `prices_included=true` and non-empty `services[]` / `services_tsv`. **Never** use `clients_pack` or summary-only responses for unit prices.
8. **Stale tools:** If catalog/price tools are missing from the host registry, tell the user to **reconnect / re-add** Tenant MCP — do not invent rates.
9. **Permissions:** Respect API errors for missing `can_*` or disabled modules (e.g. projects 403); tell the user which flag/module is needed.

If MCP tools are unavailable, say so and do not pretend the action succeeded.

## Persistence rule (required fields)

**Do not call create/assign/status until the checklist for that flow is Ready.**

- Keep a short **missing-fields** list while gathering.
- Ask gaps in **batches of at most 3** questions.
- After Ready, show a one-line plan and wait for confirm.

## Typical flows

### A — Ticket (incident / request)

1. `search` or `clients_resolve(q=…)` → pick client UUID.
2. Gather: title, description, priority, category, optional location.
3. Confirm → `tickets_create(…, client=…)`.
4. Optional: `assignment_recommend` → confirm → `tickets_assign` or `assignment_auto_assign`.
5. Updates: `tickets_set_status` (with comment if pending) / `tickets_add_comment`.

### B — Project (planned delivery) — same catalog pattern as Client MCP

1. `clients_resolve` → `client_id`.
2. **`clients_service_prices(client_id)`** (page if `has_more`) — required before activities.
3. Confirm → `projects_create(name, client=…)` (do not invent `default_hourly_rate`).
4. **`project_activities_create(project=…, service_rate_id=<catalog id>)`** — **required** `service_rate_id` (catalog seeds name/hours/rate); optional staff `hourly_rate` override.
5. Optional materials → `projects_set_offer_validity` → `projects_offer_pdf`.
6. Mark activities done via `project_activities_update(status=done, …)`.

### C — Reports / billing pack

1. Resolve client if needed.
2. `reports_list` → `reports_run(report_id, period=…, client_id=…)`.
3. Invoices: `invoices_list` / `invoices_by_month` / `invoices_pdf` when `can_read_invoices`.

### D — Client workspace snapshot

1. `clients_resolve` → `clients_pack(client_id)` (projects + tickets + invoices-by-month + **catalog summary only**).

### E — Service catalog / unit prices (per client SLA)

1. `clients_resolve(q=…)` → `client_id`.
2. **`clients_service_prices(client_id)`** (preferred) or `clients_service_catalog(client_id)`.
3. Confirm `prices_included=true` and `services[]` (or `services_tsv`) has lines with `rate` / `eu`.
4. If `has_more`, repeat with `offset=next_offset` (default page size 80) or pass `limit=250`.
5. Use line `id` as `service_rate_id` on tickets / activities. Do **not** invent emergency/surcharge rates unless a catalog line exists (`q=` search).

## Presentation when unsure

```
Recommended: <action>
Why: <one sentence>
Need from you: <missing fields or confirm>
```

## Not this skill

- Client portal / Hosted Client MCP (`ck_live_` / `/mcp`) → use **Stadrustech-MCP** / `itsm-client-mcp-agent`
- Changing product scope / RBAC design → platform maintainers only
