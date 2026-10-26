# Hosted Tenant MCP — tools

Authenticate every request with `X-API-Key: sk_live_…` (or Bearer). All data is scoped to that key’s **tenant**.

MCP URL: `https://ticket.scardustech.com/tenant-mcp`  
Never use Client MCP (`/mcp`) or `ck_live_` keys.

## Staff rules

- Prefer **`search`** / **`clients_resolve`** by name before UUIDs.
- Confirm with the user before create / assign / bulk / status writes.
- `pending` status **requires** a comment (`tickets_set_status`).
- Assignment uses Django **user PK** (`user_id` from technicians list).

## Typical flows

**Ticket:** `clients_resolve` → `clients_services(client_id)` → `tickets_create(service_rate_id=…)` → `assignment_recommend` / `tickets_assign` → `tickets_set_status` / `tickets_add_comment`

**Project:** `projects_create` → `clients_services(client_id)` → `project_activities_create(service_rate_id=…)` → `projects_set_offer_validity` → `projects_offer_pdf`

**Reports:** `reports_list` → `reports_run(report_id, period=…, client_id=…)`

**Client pack:** `clients_resolve` → `clients_pack` (includes that client’s `services` prices)

**Service prices (per client):** `clients_resolve` → `clients_services(client_id)` — preferred. Or `services_list(client=…)`; omit client → `by_client[]` (one block per org).

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

### Discovery / clients / invoices / services

| Tool | Purpose |
|------|---------|
| `search` | Clients/projects/tickets/invoices by name |
| `clients_list` / `clients_get` / `clients_resolve` | Client lookup |
| `clients_pack` | Workspace pack (+ that client’s SLA prices) |
| `clients_services` | **Preferred** — SLA prices for one client only |
| `services_list` / `services_get` | Per-client prices (`client=`) or `by_client[]` |
| `invoices_list` / `invoices_get` / `invoices_summary` / `invoices_by_month` | Billing ledger |
| `invoices_pdf` | Invoice PDF (base64) |

### Projects

| Tool | Purpose |
|------|---------|
| `projects_list` / `projects_get` / `projects_create` / `projects_update` | Projects |
| `projects_detail` / `projects_progress` | Detail / % complete |
| `projects_set_offer_validity` | Offer validity days |
| `projects_offer_pdf` / `_html` / `_xlsx` | Commercial offer export |
| `project_activities_list` / `_create` / `_update` | Activities |
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
