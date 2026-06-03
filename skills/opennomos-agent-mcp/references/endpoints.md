# OpenNomos MCP Endpoints

Use this file when you need the exact route table or a ready-to-run curl example.

## Contents

- [Authentication](#authentication)
- [Route Table](#route-table)
- [Minimal Curl Examples](#minimal-curl-examples)
- [Project Discovery Details](#project-discovery-details)
- [Event Stream Validation](#event-stream-validation)
- [Points And Ledger Interpretation](#points-and-ledger-interpretation)
- [Core Tool Set](#core-tool-set)
- [Interpretation Rules](#interpretation-rules)

## Authentication

All protected requests use:

```http
Authorization: Bearer <token>
```

`<token>` should be an MCP API key that starts with `nk_`.

Practical rule:

- if you only have `BASE_URL + nk_ key`, you can use the MCP data tools below
- if the key is not in the user message, first check shell `OPENNOMOS_MCP_KEY`, then repo-root `.env`
- never print the key; redact `nk_` values in logs and final answers

Default base URL:

```bash
export BASE_URL="${OPENNOMOS_MCP_BASE_URL:-https://api.opennomos.com}"
```

Default key resolution:

```bash
export TOKEN="${OPENNOMOS_MCP_KEY:-}"
```

When running inside a repo that has `.env`, shell values override `.env`. Read `.env` directly only to resolve missing values; do not write secrets into markdown or logs.

For localhost, prefer:

```bash
curl --noproxy '*'
```

## Route Table

## MCP Data Tools

| Tool capability | Method | Path | Auth | Notes | Common params |
| --- | --- | --- | --- | --- | --- |
| List accessible projects | `GET` | `/api/v1/mcp/projects` | `nk_` key | Returns project and application records | `page`, `page_size` |
| List public/explorable projects | `GET` | `/api/v1/mcp/explore/projects` | `nk_` key | Usually ordered by trend or rank | `page`, `limit` |
| Get project detail | `GET` | `/api/v1/mcp/projects/:project_id` | `nk_` key | Single project base info | `project_id` |
| Get project tasks | `GET` | `/api/v1/mcp/projects/:project_id/tasks` | `nk_` key | `404` usually means no configured task rule | `project_id` |
| Get project overview | `GET` | `/api/v1/mcp/projects/:project_id/overview` | `nk_` key | KPI/overview data | `project_id`, `window=7d|30d` |
| Get project daily ops | `GET` | `/api/v1/mcp/projects/:project_id/daily-ops` | `nk_` key | Daily registration and usage metrics from `user_signup` and `daily_use` aggregates | `project_id`, `window=7d|30d` |
| Get my project event stream | `GET` | `/api/v1/mcp/projects/:project_id/event-stream` | `nk_` key | Only returns the current authenticated user's recent raw events for this project; use this to check whether my task event was ingested | `project_id`, `limit` |
| Get my points overview | `GET` | `/api/v1/mcp/me/points` | `nk_` key | Total points, active projects, per-project balances | None |
| Get my project points | `GET` | `/api/v1/mcp/me/projects/:project_id/points` | `nk_` key | Points for one project | `project_id` |
| Get my project ledger | `GET` | `/api/v1/mcp/me/projects/:project_id/ledger` | `nk_` key | Accounting-level ledger | `project_id`, `page`, `limit`, `entry_type`, `date_from`, `date_to` |

## Minimal Curl Examples

Assume:

```bash
export BASE_URL="${OPENNOMOS_MCP_BASE_URL:-https://api.opennomos.com}"
export TOKEN="${OPENNOMOS_MCP_KEY:-}"
```

If `TOKEN` is empty, ask the user to provide an `nk_` key before making live calls.

### Connectivity

```bash
curl -sS -i "$BASE_URL/health"
```

For localhost only:

```bash
curl --noproxy '*' -sS -i "$BASE_URL/health"
```

### Validate token or key

```bash
curl -sS -i \
  -H "Authorization: Bearer $TOKEN" \
  "$BASE_URL/api/v1/mcp/me/points"
```

Treat this response as current actor evidence before comparing event streams.

### List accessible projects

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  "$BASE_URL/api/v1/mcp/projects?page=1&page_size=20"
```

### List public or platform projects

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  "$BASE_URL/api/v1/mcp/explore/projects?page=1&limit=20"
```

### Get tasks for one project

```bash
curl -sS -i \
  -H "Authorization: Bearer $TOKEN" \
  "$BASE_URL/api/v1/mcp/projects/<project_id>/tasks"
```

### Get daily operational metrics for one project

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  "$BASE_URL/api/v1/mcp/projects/<project_id>/daily-ops?window=7d"
```

Response interpretation:

- `registered_users`: unique users with `user_signup` on that date
- `daily_use_users`: unique users with `daily_use` on that date
- `active_users`: currently the same value as `daily_use_users`
- `signup_events`: total `user_signup` event count
- `daily_use_events`: total `daily_use` event count

### Check whether my task event was recorded

```bash
curl -sS -i \
  -H "Authorization: Bearer $TOKEN" \
  "$BASE_URL/api/v1/mcp/projects/<project_id>/event-stream?limit=50"
```

### Get my ledger for one project

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  "$BASE_URL/api/v1/mcp/me/projects/<project_id>/ledger?page=1&limit=20"
```

## Project Discovery Details

For a full project inventory:

1. Call `/api/v1/mcp/projects?page=1&page_size=20`.
2. Call `/api/v1/mcp/explore/projects?page=1&limit=20`.
3. If the response includes `total_pages`, fetch remaining pages when the user asks for a complete list.
4. Normalize IDs before per-project calls:
   - `/projects`: use `.data.items[].project_id`; fall back to `.id` only when needed.
   - `/explore/projects`: use `.data.items[].id`; fall back to `.project_id` only when needed.
5. De-duplicate project IDs.
6. Call `/api/v1/mcp/projects/:project_id/tasks` for each relevant ID.
7. Treat `/tasks` `404` as "no task rule configured".

Do not rely on `/me/points` active project data as the complete project list. It is balance/actor context, not project discovery.

## Event Stream Validation

Use `/api/v1/mcp/projects/:project_id/event-stream?limit=50` to validate task ingestion for the current authenticated user.

Before comparing events:

1. Confirm the current actor with `/api/v1/mcp/me/points`.
2. Know the action time or submission timestamp when available.
3. Match on the strongest available signal:
   - `event_type`
   - `event_id`
   - task-specific payload fields
   - public URL or contribution URL inside payload preview
   - timestamp after the action was performed

Use these outcomes:

- `recorded`: event-stream shows a confident matching event.
- `pending`: action completed, but no matching event is visible yet.
- `unclear`: nearby events exist, but the match is not confident.
- `failed`: no evidence appears after a reasonable wait, or the platform rejected the action.

Intermediate evidence such as tweet URLs, form success messages, product-side usage API responses, or browser clicks is useful context, but event-stream is the final task-validation source unless the user explicitly asks for points or ledger.

## Points And Ledger Interpretation

Use points and ledger endpoints for accounting questions, not as the primary task-validation source.

- `/api/v1/mcp/me/points`: confirms the current actor context and high-level balances.
- `/api/v1/mcp/me/projects/:project_id/points`: confirms one project's current balance for the authenticated user.
- `/api/v1/mcp/me/projects/:project_id/ledger`: confirms accounting entries and may require pagination, `date_from`, `date_to`, or `entry_type` filters.

Interpretation rules:

- Group ledger entries by `type`.
- Treat `daily_emission`, `rollback`, and similar values as accounting categories, not guaranteed task names.
- Only attribute points to a task when ledger details or nearby event-stream evidence make that link explicit.
- If the ledger confirms a balance movement but not the source task, say that directly.

## Core Tool Set

These are the routes this skill should focus on:

- `GET /api/v1/mcp/projects`
- `GET /api/v1/mcp/explore/projects`
- `GET /api/v1/mcp/projects/:project_id`
- `GET /api/v1/mcp/projects/:project_id/tasks`
- `GET /api/v1/mcp/projects/:project_id/overview`
- `GET /api/v1/mcp/projects/:project_id/daily-ops`
- `GET /api/v1/mcp/projects/:project_id/event-stream`
- `GET /api/v1/mcp/me/points`
- `GET /api/v1/mcp/me/projects/:project_id/points`
- `GET /api/v1/mcp/me/projects/:project_id/ledger`

## Interpretation Rules

- Use `/api/v1/mcp/projects` for “what can this user access?”
- Use `/api/v1/mcp/explore/projects` for “what exists on the platform?”
- Treat `/tasks` `404` as “no task rule configured yet”
- Use `/api/v1/mcp/projects/:project_id/daily-ops` for project registration and daily-use trends
- Interpret `daily-ops.active_users` as `daily_use_users` unless the backend later introduces a separate active-user event
- Use `/api/v1/mcp/projects/:project_id/event-stream` for “was my task event recorded?”
- Report event-stream validation as `recorded`, `pending`, `unclear`, or `failed`
- Treat ledger `type` values as accounting categories, not guaranteed task names
- Do not require points or ledger changes when the question is only “was my task event recorded?”
- If the user asks for current state, query live data every time
