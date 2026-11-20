# Hosted Tenant MCP — tools

Authenticate every request with `X-API-Key: sk_live_…` (or Bearer). All data is scoped to that key’s **tenant**.

MCP URL: `https://ticket.scardustech.com/tenant-mcp`  
Never use Client MCP (`/mcp`) or `ck_live_` keys.

## Staff rules

- Prefer **`search`** / **`clients_resolve`** by name before UUIDs.
- Confirm with the user before create / assign / bulk / status writes.
- `pending` status **requires** a comment (`tickets_set_status`).
- Assignment uses Django **user PK** (`user_id` from technicians list).
- **Exact prices / service offers:** use **`clients_service_prices`** (or alias **`clients_offer_prices`**) for that client’s EUR/h — never `clients_pack` summaries.
- **Retainers / recurring:** do **not** `projects_create` — draft a document in chat, create recurring charge in web **Finance & Accounting → Finance → Recurring charges**.
- After server updates, **reconnect** the MCP connector so new tools appear (hosts cache registries).

## Typical flows

**HARD RULE — invoice lines:** Tenant MCP never adds ticket/project invoice lines. Backend auto-adds on ticket resolve/close and on `projects_complete` (draft proforma). `invoices_*` = read/email/reminders only.

**Ticket (open / assign / resolve with retainer check):** `clients_resolve` → optional `clients_service_prices` → `tickets_create(client=…, priority=…)` → `assignment_list_technicians` / `assignment_recommend` → `tickets_assign` → `tickets_update` / `tickets_add_comment` as needed → `clients_retainer_hours` / `tickets_billing(hours=…)` → confirm → `tickets_resolve(hours=…, service_rate_id=?, billable_note=work name)`

**Monthly retainer / recurring (not a project):** `clients_resolve` → `clients_service_prices(client_id)` → draft plan document in the AI chat → user creates recurring charge in web **Finance & Accounting → Finance → Recurring charges**

**Consult exact prices / draft a service offer in chat:** `clients_resolve` → **`clients_service_prices(client_id)`** (or `clients_offer_prices`) → quote `services[].rate` for that client only

**One-time project + commercial offer PDF:** `clients_resolve` → `clients_service_prices(client_id)` → **`projects_create(client=…, activities=[{service_rate_id,…}])`** (≥1 activity required in create; `description` = short objective only — no checklist dumps) → optional extras / materials → `projects_set_offer_validity` → `projects_offer_pdf`

**Approve client-submitted project:** `projects_approve(project_id)` when `status=submitted` (or web Project List → **Approve**)

**Complete delivery project:** `projects_complete(project_id)` (or web **Mark complete**) — plan hours/rates → proforma

**Reports:** `reports_list` → `reports_run(report_id, period=…, client_id=…)`

**Invoice email + reminders:** `invoices_list` / `invoices_get` → confirm → `invoices_send_email(invoice_id)` (resend uses the same tool) → after `email_sent_at`, `invoices_set_reminders(invoice_id, enabled=true, interval_days=15)`. Reminders stay locked until that first send.

**Client pack:** `clients_resolve` → `clients_pack` (workspace snapshot; **catalog summary only** — no unit prices)

**Client Process questionnaires (site assessment / onboarding):** This **is** how ITSM stores pre-contract assessments on the client record (Processes tab) — not tickets/projects and not chat-only notes. `clients_resolve` → `clients_processes_list` → `clients_process_start` (`site_assessment` \| `client_onboarding`, or continue `active_run_id`) → `clients_process_get(unanswered_only=true)` → ask **platform** questions → `clients_process_update(items=[{id,answer,notes}])` (extra detail → `notes`) → `clients_process_complete`. If tools missing → reconnect Tenant MCP. Never invent answers or store live secrets.

**Credential vault stubs (planned):** When assessment finds a server with credentials, `clients_vault_stubs_create` / `clients_process_vault_stubs_create` create **empty** placeholders (title/kind/host only). MCP **never** accepts passwords — operators populate secrets in web Client → Vault. Optional `clients_vault_stubs_list` (metadata only).

**Non-principal org people (per client, when needed):** `people_template(client_id)` → confirm → `people_create` or `people_onboard` (optional asset/service links). List/update/offboard: `people_list` / `people_get` / `people_update` / `people_offboard`. These are **not** portal users — never mix with Principal Users. Needs `assets` module.

**Exact service prices (per client SLA):** `clients_resolve` → **`clients_service_prices(client_id)`** / **`clients_offer_prices(client_id)`**  
Expect `prices_included=true`, `services[]` with `rate` (**exact client EUR/h to quote**), `eu` (EU list reference), `id`, `code`, `name`, plus `services_tsv`. Default page `limit=80`; if `has_more`, call with `offset=next_offset` (or `limit=250`).  
Other aliases: `clients_service_catalog`, `clients_services`. `services_list(client=…)` same; `services_list()` with no client → summaries only.

## Tool list

### Tickets

| Tool | Purpose |
|------|---------|
| `tickets_list` | List/filter; **`status=opened`** = open+in_progress+pending (not resolved/closed) |
| `tickets_get` | Detail + SLA |
| `tickets_create` | **Open** ticket for a client (prefer `client` UUID) |
| `tickets_update` | **Priority** and other PATCH fields (not preferred for status/assign) |
| `tickets_assign` | **Assign** by `technician_id` (Django user PK) → in_progress |
| `tickets_set_status` | **Status** with SLA pause/resume (`pending` needs comment) |
| `tickets_billing` | Preview SLA/retainer bill decision for a ticket |
| `tickets_resolve` | Resolve with hours + retainer-aware bill/€0 line |
| `clients_retainer_hours` | Current-month retainer pool (used/remaining) |
| `tickets_recalculate_sla` | Recalc due dates after priority/client change |
| `tickets_bulk` | Bulk status/priority/assignee (no per-ticket SLA pause) |
| `tickets_list_comments` | List comments |
| `tickets_add_comment` | **Comment** (`is_internal_note` for staff-only) |

### Assignment

| Tool | Purpose |
|------|---------|
| `assignment_list_technicians` | Techs + workload; use `user_id` |
| `assignment_list_locations` | Locations |
| `assignment_recommend` | Score technicians (no assign) |
| `assignment_auto_assign` | Assign best match |

### Discovery / clients / invoices / service prices

| Tool | Purpose |
|------|---------|
| `search` | Clients/projects/tickets/invoices by name |
| `clients_list` / `clients_get` / `clients_resolve` | Client lookup |
| `clients_pack` | Workspace pack (**catalog SUMMARY only** — not for quotes) |
| `clients_service_prices` | **PRIMARY — exact client prices for offers/quotes** (`rate`/`eu`/`id`/`name` + TSV) |
| `clients_offer_prices` | Alias of `clients_service_prices` (offer-oriented name) |
| `clients_processes_list` | Client Process questionnaires on the client (`site_assessment`, `client_onboarding`) |
| `clients_process_start` | Start questionnaire on client record (`definition_key`) — not a ticket/project |
| `clients_process_get` | Read questionnaire / `unanswered_only` interview loop |
| `clients_process_update` | **Save** `{items:[{id,answer,notes}], date, assessor}` — overflow facts in `notes` |
| `clients_process_complete` | Mark questionnaire completed |
| `clients_process_abandon` | Abandon in-progress run |
| `people_template` / `people_list` / `people_get` | Non-principal org people field guide / list / detail |
| `people_create` / `people_update` | Create or patch a non-portal org person for a client |
| `people_onboard` / `people_offboard` | Create + link assets/services; soft-deactivate with checklist |
| `non_principal_users_*` | Aliases of `people_*` (UI: Non-principal Users) |
| `clients_service_catalog` | Full priced catalog page (default limit 80; page via offset) |
| `clients_services` | Alias of `clients_service_catalog` |
| `services_list` / `services_get` | `client=` → priced; no client → `by_client[]` summaries |
| `invoices_list` / `invoices_get` / `invoices_summary` / `invoices_by_month` | Finance & Accounting → Accounting (client AR) |
| `invoices_pdf` | Invoice PDF (base64) |
| `invoices_send_email` | **Email / resend** FATURE PDF (`to`/`cc` optional, comma-separated) |
| `invoices_set_reminders` | Per-invoice reminder opt-in + `interval_days` (only after first send) |

### Projects

| Tool | Purpose |
|------|---------|
| `projects_list` / `projects_get` / `projects_create` / `projects_update` | Delivery projects (not retainers); **create requires `activities[]`** |
| `projects_approve` | Client-submitted → planned |
| `projects_complete` | Mark complete; apply plan hours/rates → proforma |
| `projects_detail` / `projects_progress` | Detail / % complete |
| `projects_set_offer_validity` | Offer validity days |
| `projects_offer_pdf` / `_html` / `_xlsx` | Commercial offer export |
| `project_activities_list` / `_create` / `_update` | Activities — **create requires `service_rate_id` from catalog** |
| `project_materials_list` / `_create` / `_update` | Materials/BOM |

### GRC (module `grc`, flags `can_read_grc` / `can_write_grc`)

Canonical spine: `GRC_MODULE.md` §2.2 — obligation + **one enclave** (in/out assets) + work **contributions** → SSP/KPI.

| Tool | Purpose |
|------|---------|
| `grc_frameworks_list` | Published framework packs |
| `grc_assignment_create` / `grc_assignment_update` | Assign pack; target date / notes; planned **active vs watch** |
| `grc_implementations_list` / `grc_implementation_update` | Control status/notes + planned dates |
| `grc_control_gantt` / `grc_gaps` / `grc_dashboard` / `grc_ssp_*` | Gaps, schedule, SSP |
| `grc_obligation_*` | Contractual required-control baselines (L1 then graduate L2) |
| **Planned** `grc_enclave_*` | Upsert enclave; list members; **move assets in↔out**; undeclared; bulk set |
| **Planned** `compliance_contribution_*` / create-work `compliance_controls[]` | Stamp ticket/project → control + enclave for KPI |
| **Planned** `grc_compliance_kpi` | Required %, work-backed, undeclared, affirmation-ready |

Reconnect Tenant MCP after deploy so new tools appear.

### Reports

| Tool | Purpose |
|------|---------|
| `reports_list` | Available report ids |
| `reports_schema` | Schema for one report |
| `reports_run` | Run report (`period` **or** `from_date`/`to_date`, optional `client_id`) |
| `reports_tickets_hours` | Alias for `tickets.hours` — weekly hours + billable split |

Allowed `report_id`: `tickets.summary`, `tickets.volume`, `tickets.hours`, `sla.compliance`, `clients.overview`, `projects.hours`, `billing.invoices`, `risks.heatmap`.

**Weekly hours pack (`tickets.hours` / `reports_tickets_hours`):**

- Prefer `from_date`/`to_date` when the report week is not MCP Monday→today (e.g. Friday Berlin cutover).
- Else `period=week` is fine.
- Totals: `hours_logged`, `hours_billable`, `hours_non_billable`.
- Rows: per-ticket `hours_logged`, `billing_status` (`billable`|`non_billable`|`hours_not_logged`).
- Clients: `hours_remaining` + `retainer_hours` (current-month pool when on hours SLA).
- Fallback: `tickets_get` → `hours_logged` / `billing_status` / `billing.retainer_hours`.

## String references (same text the hosted FastMCP loads)

- [`references/instructions.txt`](../references/instructions.txt) — server `instructions` (auth, SLA, flows, enums)
- [`references/workflows.txt`](../references/workflows.txt) — per-tool field / workflow playbook

Monorepo mirror: `itsm/mcp/tenant-mcp/references/` + host config scripts in `itsm/mcp/tenant-mcp/configs/`.

## Common errors

| Code | Meaning |
|------|---------|
| 401 | Missing / invalid key |
| 403 | Missing permission / module disabled |
| 404 | Not found in this tenant |
| 400 | Validation |
