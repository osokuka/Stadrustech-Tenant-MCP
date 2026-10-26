# Hosted Tenant MCP — tools

Authenticate every request with `X-API-Key: sk_live_…` (or Bearer). All data is scoped to that key’s **tenant**.

MCP URL: `https://ticket.scardustech.com/tenant-mcp`  
Never use Client MCP (`/mcp`) or `ck_live_` keys.

## Staff rules

- Prefer **`search`** / **`clients_resolve`** by name before UUIDs.
- Confirm with the user before create / assign / bulk / status writes.
- `pending` status **requires** a comment (`tickets_set_status`).
- Assignment uses Django **user PK** (`user_id` from technicians list).
- **Unit prices:** use **`clients_service_prices`** — never `clients_pack` summaries.
- After server updates, **reconnect** the MCP connector so new tools appear (hosts cache registries).

## Typical flows

**Ticket:** `clients_resolve` → `clients_service_prices(client_id)` → `tickets_create(service_rate_id=…)` → `assignment_recommend` / `tickets_assign` → `tickets_set_status` / `tickets_add_comment`

**Project / offer (same as Client MCP catalog flow):** `clients_resolve` → `clients_service_prices(client_id)` → `projects_create(client=…)` → **`project_activities_create(project=…, service_rate_id=<catalog id>)`** (required) → optional materials → `projects_set_offer_validity` → `projects_offer_pdf`

**Reports:** `reports_list` → `reports_run(report_id, period=…, client_id=…)`

**Client pack:** `clients_resolve` → `clients_pack` (workspace snapshot; **catalog summary only** — no unit prices)

**Service prices (per client SLA):** `clients_resolve` → **`clients_service_prices(client_id)`**  
Expect `prices_included=true`, `services[]` with `rate` (client EUR/h), `eu` (EU list), `id`, `code`, `name`, plus `services_tsv`. Default page `limit=80`; if `has_more`, call with `offset=next_offset` (or `limit=250`).  
Aliases: `clients_service_catalog`, `clients_services`. `services_list(client=…)` same; `services_list()` with no client → summaries only.

## Tool list

### Tickets

| Tool | Purpose |
|------|---------|
| `tickets_list` | List/filter tickets |
| `tickets_get` | Detail + SLA |
| `tickets_create` | Create (prefer `client` UUID for SLA) |
| `tickets_update` | PATCH fields (prefer set_status/assign) |
| `tickets_assign` | Assign by `technician_id` (user PK) → in_progress |
| `tickets_set_status` | Status with SLA pause/resume |
| `tickets_recalculate_sla` | Recalc due dates |
| `tickets_bulk` | Bulk status/priority/assignee |
| `tickets_list_comments` | List comments |
| `tickets_add_comment` | Add comment (optional internal) |

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
| `clients_service_prices` | **Required for unit prices** — compact `rate`/`eu`/`id`/`name` + TSV |
| `clients_service_catalog` | Priced catalog page (default limit 80; page via offset) |
| `clients_services` | Alias of `clients_service_catalog` |
| `services_list` / `services_get` | `client=` → priced; no client → `by_client[]` summaries |
| `invoices_list` / `invoices_get` / `invoices_summary` / `invoices_by_month` | Billing ledger |
| `invoices_pdf` | Invoice PDF (base64) |

### Projects

| Tool | Purpose |
|------|---------|
| `projects_list` / `projects_get` / `projects_create` / `projects_update` | Projects |
| `projects_detail` / `projects_progress` | Detail / % complete |
| `projects_set_offer_validity` | Offer validity days |
| `projects_offer_pdf` / `_html` / `_xlsx` | Commercial offer export |
| `project_activities_list` / `_create` / `_update` | Activities — **create requires `service_rate_id` from catalog** |
| `project_materials_list` / `_create` / `_update` | Materials/BOM |

### Reports

| Tool | Purpose |
|------|---------|
| `reports_list` | Available report ids |
| `reports_schema` | Schema for one report |
| `reports_run` | Run report |

Allowed `report_id`: `tickets.summary`, `tickets.volume`, `sla.compliance`, `clients.overview`, `projects.hours`, `billing.invoices`.

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
