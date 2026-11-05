---
name: itsm-tenant-mcp-agent
description: >-
  Drives Scardus Tech Hosted Tenant MCP (sk_live_) tools for tenant staff:
  search clients, site assessment / client process questionnaires
  (clients_process_*), per-client SLA service prices/catalog, tickets,
  projects, invoices, and reports. Never uses ck_live_ or Client MCP.
  Persistently gathers required fields and confirms before writes.
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
7. **Exact prices / service offers:** For “what do we charge client X?” or drafting an offer, call **`clients_service_prices(client_id)`** (alias **`clients_offer_prices`**). Quote `services[].rate` (that client’s EUR/h). Require `prices_included=true` and non-empty `services[]` / `services_tsv`. **Never** use `clients_pack` or summary-only responses for unit prices.
8. **Retainers / recurring (critical):** Do **not** call `projects_create` for monthly retainers or recurring payments. Use catalog tools to **draft a document in this AI chat**, then tell staff to create the recurring charge in the **web console** (**Finance & Accounting → Finance → Recurring charges**, pick the client). One-time delivery projects only via `projects_create`. Clients may still create draft projects via Client MCP.
9. **Stale tools:** If catalog/price tools are missing from the host registry, tell the user to **reconnect / re-add** Tenant MCP — do not invent rates.
10. **Permissions:** Respect API errors for missing `can_*` or disabled modules (e.g. projects 403); tell the user which flag/module is needed.
11. **Invoice email / reminders:** `invoices_send_email` then (only after that send) `invoices_set_reminders`. Never enable reminders on an unsent invoice.

If MCP tools are unavailable, say so and do not pretend the action succeeded.

## Persistence rule (required fields)

**Do not call create/assign/status until the checklist for that flow is Ready.**

- Keep a short **missing-fields** list while gathering.
- Ask gaps in **batches of at most 3** questions.
- After Ready, show a one-line plan and wait for confirm.

## Typical flows

### 0 — Consult exact prices / draft a service offer in chat

1. `clients_resolve(q=client name)` → `client_id`.
2. **`clients_service_prices(client_id)`** or **`clients_offer_prices(client_id)`** (page if `has_more`).
3. Answer with each line’s **`rate`** (exact client EUR/h); mention `eu` only as list reference.
4. If they want a formal PDF offer: continue with flow **C** (project + activities + `projects_offer_pdf`).

### A0 — “Any opened tickets?” (reporting / triage)

1. User means **not resolved and not closed** → `tickets_list(status="opened")` (aliases: `unresolved`, `active`).
2. Optional: `clients_resolve` then `tickets_list(status="opened", client=…)`.
3. Do **not** only pass `status="open"` (that misses `in_progress` and `pending`).

### A — Ticket (open / assign / status / priority / comment)

1. `search` or `clients_resolve(q=…)` → pick client UUID.
2. Gather: title, description, priority, category, optional location / `service_rate_id`.
3. Confirm → `tickets_create(…, client=…, priority=…)`.
4. Assign: `assignment_list_technicians` or `assignment_recommend` → confirm → `tickets_assign(technician_id=user_id)` or `assignment_auto_assign`.
5. Priority change: `tickets_update(ticket_id, priority=low|medium|high|critical)`.
6. Status: `tickets_set_status` (non-empty `comment` required if `pending`; prefer resolved→closed).
7. Comment: `tickets_add_comment` (`is_internal_note=true` for staff-only).

### B — Monthly retainer / recurring payment (NOT a project)

1. `clients_resolve` → `clients_service_prices(client_id)`.
2. Draft a planning document **in this chat** (scope, included hours, monthly amount, start date).
3. **Do not** call `projects_create`. Tell the user to open the web console → **Finance & Accounting → Finance → Recurring charges** and create the recurring charge for that client.
4. Optional: point at SLA retainer fields already on the client’s assigned SLA if relevant.

### C — One-time / delivery project only

1. Confirm the work is a finite delivery (not ongoing retainer).
2. `clients_resolve` → `client_id`.
3. **`clients_service_prices(client_id)`** (page if `has_more`) — required before activities.
4. Confirm → `projects_create(name, client=…)` (do not invent `default_hourly_rate`; prefer `time_and_materials` / `fixed_price`, not `retainer`).
5. **`project_activities_create(project=…, service_rate_id=<catalog id>)`** — **required** `service_rate_id`; optional staff `hourly_rate` override.
6. Optional materials → `projects_set_offer_validity` → `projects_offer_pdf`.
7. Client-submitted awaiting review: confirm → `projects_approve(project_id)` (`submitted` → `planned`), or Approve in web Project List.
8. When delivery is done: confirm → `projects_complete(project_id)` (or web **Mark complete**) — applies plan hours/rates, then proforma.

### D — Reports / billing pack

1. Resolve client if needed.
2. `reports_list` → `reports_run(report_id, period=…, client_id=…)`.
3. Invoices: `invoices_list` / `invoices_by_month` / `invoices_pdf` when `can_read_invoices`.
4. Send/resend: confirm → `invoices_send_email(invoice_id)` (optional `to`/`cc` comma-separated).
5. Reminders (after `email_sent_at` only): confirm → `invoices_set_reminders(invoice_id, enabled=true, interval_days=15)`.

### E — Client workspace snapshot

1. `clients_resolve` → `clients_pack(client_id)` (projects + tickets + invoices-by-month + **catalog summary only**).

### E2 — Client Process questionnaires (site assessment / onboarding)

**These tools ARE the ITSM save path** for pre-contract site assessments and
onboarding checklists (Client → Processes). Do **not** say the platform cannot
store questionnaires. Do **not** create tickets or projects for assessments.
Do **not** keep answers as chat-only “assessment notes”.

Run as a **guided interview** — human answers; you persist via tools.

1. `clients_resolve(q=…)` → `clients_processes_list(client_id)`.
2. Pick the right key — **do not mix them up:**
   - **Site visit / initial IT assessment / pre-contract / uploaded assessment MD** → `definition_key=site_assessment` (**3 pages**).
   - **Post-win stand-up only** → `client_onboarding` (7 pages). Never use onboarding for a site assessment.
3. `clients_process_start(…)` **or** continue `active_run_id`. There is **no** document-upload tool — map the assessment into platform items; overflow → `notes`.
4. `clients_process_get(run_id, unanswered_only=true)` — ask **platform** questions only (do not invent a parallel form).
5. Guide through unanswered items (**any order** / batch / resume OK).
6. After each answer/batch: `clients_process_update(items=[{id, answer, notes?}])`; optional `date` / `assessor`.
   - `answer` = client’s answer to that item.
   - `notes` = observations, verification, or **extra facts with no dedicated field** (locations summary, fleet GPS, SIM provider, etc.).
7. Until `unanswered_count` is 0 (or user pauses). Confirm → `clients_process_complete` (or `clients_process_abandon`).
8. If `clients_process_*` tools are missing → **reconnect Tenant MCP**; do **not** say ITSM cannot store assessments. If a call 403s → report it; never pretend ITSM was updated.
9. Never store live passwords/API keys — vault metadata only.
10. Credentials found → empty stubs via `clients_vault_stubs_create` / `clients_process_vault_stubs_create` (labels only). Operator fills secrets in web Client → Vault.
11. Do not invent answers; ask clearly and wait.

### F — Service catalog / unit prices (per client SLA)

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
