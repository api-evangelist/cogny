---
name: Generate and stream a growth report
description: Create an AI growth report over a connected warehouse and stream results, then read the finished content.
api: openapi/cogny-openapi.yml
method: generated
operations: [listWarehouses, createReport, streamReport, getReportContent]
---

# Generate and stream a Cogny growth report

Use the Cogny Report Builder to run an AI analysis over a connected data
warehouse and collect the insights, visualizations, and recommendations.

## Auth
- Base URL: `https://api.cogny.com/v1`
- Send `Authorization: Bearer <key>` (use an `sk_test_` key while developing, `sk_live_` in production; MCP-issued `cogny_lite_` keys also work).
- Requires scopes `warehouses:read` and `reports:read` / `reports:write`.

## Steps
1. **Find a warehouse** — `GET /warehouses` (`listWarehouses`). Pick the `id` of a connected warehouse.
2. **Create the report** — `POST /reports` (`createReport`) with `{ "warehouse_id": "...", "prompt": "..." }`. The `prompt` is a natural-language analysis request (max 2000 chars). Optional `context.date_range`, `options.include_sql_queries`, etc. Response returns `data.id`, `data.status: "processing"`, and `data.stream_url`.
3. **Stream progress** — `GET /reports/{report_id}/stream` (`streamReport`). This is a Server-Sent Events stream; handle event types `status`, `query`, `result`, `insight`, `visualization`, `completed`, `error`.
4. **Read the content** — once you see the `completed` event (or `GET /reports/{report_id}` shows `status: completed`), call `GET /reports/{report_id}/content` (`getReportContent`) for the full `sections`, `insights`, `recommendations`, and `queries`.

## Rules
- Poll politely: standard tier allows 100 req/min; report creation is limited (`RATE_LIMIT_EXCEEDED` on too many concurrent reports).
- Errors use `{ "success": false, "error": { "code", "message", "details" } }` — read `error.code` (e.g. `WAREHOUSE_NOT_CONNECTED`, `INVALID_PROMPT`) and correct before retrying.
- No idempotency key is supported; do not blind-retry `createReport` on timeout — list recent reports first.
