# OpenNomos MCP Endpoints

Use this file when you need the exact route table or a ready-to-run curl example.

## Authentication

All protected requests use:

```http
Authorization: Bearer <token>
```

`<token>` should be an MCP API key that starts with `nk_`.

Practical rule:

- if you only have `BASE_URL + nk_ key`, you can use the MCP data tools below
- if the key is not in the user message, first check `OPENNOMOS_MCP_KEY`

Default base URL:

```bash
export BASE_URL="${OPENNOMOS_MCP_BASE_URL:-https://api.opennomos.com}"
```

Default key resolution:

```bash
export TOKEN="${OPENNOMOS_MCP_KEY:-}"
```

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

### Get my ledger for one project

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  "$BASE_URL/api/v1/mcp/me/projects/<project_id>/ledger?page=1&limit=20"
```

## Core Tool Set With An `nk_` Key

These are the routes this skill should focus on:

- `GET /api/v1/mcp/projects`
- `GET /api/v1/mcp/explore/projects`
- `GET /api/v1/mcp/projects/:project_id`
- `GET /api/v1/mcp/projects/:project_id/tasks`
- `GET /api/v1/mcp/projects/:project_id/overview`
- `GET /api/v1/mcp/me/points`
- `GET /api/v1/mcp/me/projects/:project_id/points`
- `GET /api/v1/mcp/me/projects/:project_id/ledger`

## Interpretation Rules

- Use `/api/v1/mcp/projects` for “what can this user access?”
- Use `/api/v1/mcp/explore/projects` for “what exists on the platform?”
- Treat `/tasks` `404` as “no task rule configured yet”
- Treat ledger `type` values as accounting categories, not guaranteed task names
- If the user asks for current state, query live data every time
