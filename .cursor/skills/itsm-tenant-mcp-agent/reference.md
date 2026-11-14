# Tenant MCP — fields, tools, workflow reference

Companion to `SKILL.md`. Read when gathering fields or choosing tools.

## Auth & scope

| Item | Rule |
|------|------|
| Key | `sk_live_…` only (`X-API-Key` or `Authorization: Bearer`) |
| Where to get it | Console **https://ticket.scardustech.com** → tenant staff → Setup **API Keys** |
| OAuth | MCP URL `/tenant-mcp` → sign in as **tenant staff**; revoke **Profile → Connections** |
| Rejected | `ck_live_…`, Client MCP `/mcp`, missing/inactive/expired key |
| Data scope | Always the key’s **Tenant** (all clients under it) |
| MCP URL | `https://ticket.scardustech.com/tenant-mcp` |
| Tool rediscovery | After server updates, **re-add / reconnect** the MCP connector (hosts cache tool lists; this chat cannot force refresh) |

## Tool map (high level)

| Intent | Tool | Notes |
|--------|------|-------|
| Find by name | `search`, `clients_resolve` | Prefer before UUIDs |
| Client pack | `clients_pack` | Projects + tickets + invoices + **catalog SUMMARY only** |
| **Client Process questionnaires** (site assessment / onboarding) | `clients_processes_list`, `clients_process_start/get/update/complete` | **ITSM save path** on client record — not tickets/projects; overflow → `notes`; reconnect if tools missing |
| **Unit prices / catalog lines** | **`clients_service_prices`** / **`clients_offer_prices`** | **Required for quotes/offers** — `rate`/`eu`/`id`/`name` + TSV |
| Full catalog page | `clients_service_catalog`, `clients_services` | Same priced lines; default `limit=80`, page via `offset` |
| Catalog index | `services_list()` no client | `by_client[]` summaries — **not** for quotes |
| “Opened tickets” (colloquial) | `tickets_list(status="opened")` | = open+in_progress+pending (not resolved/closed); aliases unresolved/active |
| Tickets CRUD-ish | `tickets_list/get/create/update` | Prefer set_status / assign for status/assignee; priority via update |
| Status | `tickets_set_status` | `pending` needs comment |
| Assign | `tickets_assign`, `assignment_*` | Use `user_id` PK |
| Comments | `tickets_list_comments`, `tickets_add_comment` | |
| Bulk | `tickets_bulk` | No per-ticket SLA pause on bulk status |
| Projects (one-time delivery) | `projects_*`, `project_activities_*`, `project_materials_*` | Not for retainers; needs projects module + flags |
| Approve client submit | `projects_approve` | `submitted` → `planned`; client drops DRAFT banner |
| Complete delivery | `projects_complete` | Plan hours/rates → completed → proforma |
| Offers | `projects_offer_pdf/html/xlsx` | Commercial offer export |
| Retainers / recurring | Draft in chat + **web Finance & Accounting → Finance → Recurring charges** | Never `projects_create` for monthly retainers |
| Invoices | `invoices_*` | Needs `can_read_invoices` |
| Invoice email / resend | `invoices_send_email` | Confirm first; unlocks reminders |
| Invoice reminders | `invoices_set_reminders` | After `email_sent_at` only; `interval_days` 1–365 |
| Reports | `reports_list` / `reports_schema` / `reports_run` | Tenant-only: `clients.overview`, `tickets.hours` |
| **Weekly hours / billable pack** | **`reports_tickets_hours`** (or `reports_run(tickets.hours)`) | Prefer `from_date`/`to_date` for custom weeks; else `period=week` |
| Ticket billing card | `tickets_get` → `hours_logged`, `billing_status`, `billing` | Detail includes `billing.retainer_hours` when on hours SLA |
| Retainer pool only | `clients_retainer_hours(client_id)` | Current-month used/remaining |

Full list: [docs/TOOLS.md](../../../docs/TOOLS.md).

## Service catalog / prices (critical)

**Never treat `clients_pack` or a summary (`prices_included=false`, empty `services[]`) as the price list.**

| Step | Tool | Expect |
|------|------|--------|
| 1 | `clients_resolve(q=…)` | `client.id` |
| 2 | **`clients_service_prices(client_id)`** (or `clients_offer_prices`) | `prices_included=true`, non-empty `services[]` / `services_tsv` |
| 3 | If `has_more` | Call again with `offset=next_offset` (default page 80) or `limit=250` |
| 4 | Quotes / tickets / activities | Use each line’s `id` as `service_rate_id`; `rate` = client EUR/h, `eu` = EU list |

Each priced line (compact): `rate` (client EUR/h), `eu` (EU list), `id`, `code`, `name`, `cat`.  
Also: `client_hourly_rate` / `hourly_rate` (= rate), `eu_list_hourly_rate` / `list_hourly_rate` (= eu).  
`catalog_list_price_percent` / `pct` = this client’s SLA discount of EU list (e.g. Eskavatori **45** → helpdesk EU 85 → **38.25**).

If the host tool list is stale (~43 tools, no `clients_service_*`), tell the user to **reconnect Tenant MCP** — do not invent rates.

## Ticket fields

| Field | Required | Notes |
|-------|----------|-------|
| title | yes | Specific |
| description | strongly preferred | Impact, systems, steps |
| client | preferred for SLA | UUID from resolve/search |
| service_rate_id | optional | From `clients_service_prices` / catalog; seeds billable_unit_price |
| billable_unit_price | optional | Staff override (wins over catalog) |
| priority | default medium | low/medium/high/critical |
| category | default incident | |
| location | optional | UUID |
| technician_id | for assign | **User PK** from `assignment_list_technicians` |

## Status playbook

| Intent | status | comment |
|--------|--------|---------|
| Reopen | `open` | why |
| Start work | `in_progress` | optional |
| Wait on customer | `pending` | **required** (SLA pause) |
| Work done | `resolved` | summary |
| Finalize | `closed` | optional |

## Project fields (delivery projects only)

| Field | Required | Notes |
|-------|----------|-------|
| name, client | yes on create | |
| billing_model | optional | Prefer `time_and_materials` / `fixed_price`. **Do not** use project create for monthly retainers — use web recurring charges |
| default_hourly_rate | optional | activities inherit if omitted |
| activity service_rate_id | **required** on create | From `clients_service_prices` — catalog seeds name/hours/rate |
| activity name, estimated_hours | optional | Defaults from catalog line |
| activity hourly_rate | optional | Staff override only (wins over catalog) |
| offer_valid_days | for offer | `projects_set_offer_validity` |
| approve | when `submitted` | `projects_approve` or web **Approve** → `planned` |
| complete | when delivery done | `projects_complete` or web **Mark complete** |

## Report ids

`tickets.summary` · `tickets.volume` · `tickets.hours` · `sla.compliance` · `clients.overview` · `projects.hours` · `billing.invoices` · `risks.heatmap`

### `tickets.hours` (weekly agent packs)

| Call | When |
|------|------|
| `reports_tickets_hours(client_id=…, from_date=…, to_date=…)` | **Preferred** when the pack week is custom (e.g. Friday Berlin cutover) |
| `reports_tickets_hours(client_id=…, period="week")` | Only if Monday→today is fine |
| `reports_run(report_id="tickets.hours", …)` | Same payload |

**Payload to use in the Word/table pack**

| Source | Fields |
|--------|--------|
| `totals` | `hours_logged`, `hours_billable`, `hours_non_billable`, `tickets_hours_not_logged` |
| `rows[]` | `hours_logged`, `billing_status` (`billable`\|`non_billable`\|`hours_not_logged`), `billing_label`, `is_billable`, `billable_note` |
| `clients[]` | `hours_remaining`, `retainer_hours` (current-month pool; not period-scoped) |

**Fallback:** `tickets_get` → top-level `hours_logged` / `billing_status` / `billing.retainer_hours`.

Period presets: `week` | `month` | `quarter` | `year` | `last_7d` | `last_30d` | `last_90d`. Explicit `from_date`/`to_date` (ISO) override presets.

## Permission flags (key)

| Flag | Needed for |
|------|------------|
| `can_read_tickets` | list/get/reports tickets; service catalog read |
| `can_create_tickets` | create |
| `can_update_tickets` | update / set_status / bulk |
| `can_assign_tickets` | assign / auto_assign |
| `can_read/create_comments` | comments |
| `can_read/create/update_projects` | projects surface; catalog read |
| `can_read_invoices` | invoices; catalog read |

## Common errors

| Code | Meaning |
|------|---------|
| 401 | Missing/invalid key |
| 403 | Missing `can_*` / module off / wrong audience |
| 404 | Not found in this tenant |
| 400 | Validation / no SLA |
