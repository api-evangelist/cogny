---
name: Triage growth tickets
description: List AI-generated growth tickets, inspect one, update its status/priority, or dismiss it with a reason.
api: openapi/cogny-openapi.yml
method: generated
operations: [listTickets, getTicket, updateTicket, dismissTicket]
---

# Triage Cogny growth tickets

Growth tickets are AI-generated marketing recommendations with impact/effort
estimates and action items. Use this flow to work a backlog.

## Auth
- Base URL: `https://api.cogny.com/v1`
- `Authorization: Bearer <key>`; requires `tickets:read` (read) and `tickets:write` (mutate).

## Steps
1. **List open tickets** — `GET /tickets` (`listTickets`), filtering by `warehouse_id`, `status`, `priority`, or `category`. Page with `limit` + `cursor` (response `pagination.next_cursor`).
2. **Inspect one** — `GET /tickets/{ticket_id}` (`getTicket`) to read `description`, `estimated_impact`, `effort`, `action_items`, `supporting_data`.
3. **Act on it** — `PATCH /tickets/{ticket_id}` (`updateTicket`) with any of `status`, `priority`, `assignee`, `notes`, `action_items`.
4. **Or dismiss it** — `POST /tickets/{ticket_id}/dismiss` (`dismissTicket`) with `reason` (one of `not_applicable`, `already_implemented`, `insufficient_impact`, `resource_constraints`, `other`) and optional `notes`.

## Rules
- For many tickets at once, use `POST /tickets/bulk-update` instead of looping `PATCH`.
- Errors follow the `{ success:false, error:{code,message,details} }` envelope; `INSUFFICIENT_PERMISSIONS` means the key lacks `tickets:write`.
- Cursor pagination only — never assume offset paging.
