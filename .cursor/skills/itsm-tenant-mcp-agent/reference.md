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

## Tool map (high level)

| Intent | Tool | Notes |
|--------|------|-------|
| Find by name | `search`, `clients_resolve` | Prefer before UUIDs |
| Client pack | `clients_pack` | Projects + tickets + invoices-by-month |
| Service prices | `services_list`, `services_get` | SLA catalog EUR/h; prefer `client=` |
| Tickets CRUD-ish | `tickets_list/get/create/update` | Prefer set_status / assign for status/assignee |
| Status | `tickets_set_status` | `pending` needs comment |
| Assign | `tickets_assign`, `assignment_*` | Use `user_id` PK |
| Comments | `tickets_list_comments`, `tickets_add_comment` | |
| Bulk | `tickets_bulk` | No per-ticket SLA pause on bulk status |
| Projects | `projects_*`, `project_activities_*`, `project_materials_*` | Needs projects module + flags |
| Offers | `projects_offer_pdf/html/xlsx` | Commercial offer export |
| Invoices | `invoices_*` | Needs `can_read_invoices` |
| Reports | `reports_list/schema/run` | Includes `clients.overview` (tenant-only) |

Full list: [docs/TOOLS.md](../../../docs/TOOLS.md).

## Ticket fields

| Field | Required | Notes |
|-------|----------|-------|
| title | yes | Specific |
| description | strongly preferred | Impact, systems, steps |
| client | preferred for SLA | UUID from resolve/search |
| service_rate_id | optional | From `services_list(client=…)`; seeds billable_unit_price |
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

## Project fields

| Field | Required | Notes |
|-------|----------|-------|
| name, client | yes on create | |
| billing_model | optional | retainer / time_and_materials / fixed_price |
| default_hourly_rate | optional | activities inherit if omitted |
| activity name, estimated_hours | preferred | status planned→done |
| activity service_rate_id | optional | From `services_list`; seeds hourly_rate |
| activity hourly_rate | optional | Staff override (wins over catalog) |
| offer_valid_days | for offer | `projects_set_offer_validity` |

## Report ids

`tickets.summary` · `tickets.volume` · `sla.compliance` · `clients.overview` · `projects.hours` · `billing.invoices`

Period: `week` | `month` | `quarter` | `year` | `last_7d` | `last_30d` | `last_90d` — or `from_date`/`to_date`.

## Permission flags (key)

| Flag | Needed for |
|------|------------|
| `can_read_tickets` | list/get/reports tickets |
| `can_create_tickets` | create |
| `can_update_tickets` | update / set_status / bulk |
| `can_assign_tickets` | assign / auto_assign |
| `can_read/create_comments` | comments |
| `can_read/create/update_projects` | projects surface |
| `can_read_invoices` | invoices |

## Common errors

| Code | Meaning |
|------|---------|
| 401 | Missing/invalid key |
| 403 | Missing `can_*` / module off / wrong audience |
| 404 | Not found in this tenant |
| 400 | Validation / no SLA |
