---
name: opennomos-agent-mcp
description: Query the OpenNomos MCP REST layer with the default base URL `https://api.opennomos.com` and an `nk_` API key. Use when tasks involve analyzing OpenNomos projects, available tasks, project overview, daily operational metrics, current actor identity, my task event stream, user points, or points ledger from the live backend.
---

# OpenNomos Agent MCP

Use this skill to inspect live OpenNomos MCP data through the current REST implementation.

Important: this skill is for the current OpenNomos MCP REST layer, not a standard MCP transport server. Treat it as authenticated HTTP API access.

## Quick Start

1. Default `BASE_URL` to `https://api.opennomos.com` unless the user explicitly provides another host.
2. Determine the `nk_` API key.
3. Resolve shell environment before repo `.env`; do not assume `.env` has been loaded into the process.
4. If no key is available after explicit input, shell env, and repo `.env`, stop and ask the user to provide one.
5. For localhost, call `curl --noproxy '*'` to avoid local proxy interference.
6. Check `GET /health` first when connectivity is unclear.
7. Call `GET /api/v1/mcp/me/points` before comparing event streams; this verifies the key and establishes the current actor context.
8. Pick the smallest set of MCP endpoints needed for the question.
9. Summarize the live result in plain language instead of dumping raw JSON.

## Inputs To Collect

- Base URL:
  - default: `https://api.opennomos.com`
  - override only when the user gives a different host such as `http://localhost:8100`
- One auth credential:
  - MCP API key starting with `nk_`

Default assumption:

- if the user only gives `BASE_URL + nk_ key`, proceed directly with MCP data routes
- if the user gives no key, check `OPENNOMOS_MCP_KEY` in the shell environment, then the repo-root `.env`
- if no key is available, ask the user for a key before live calls

If the user asks for current platform state, always query the live endpoint. Do not answer from memory.
Never print or store the `nk_` key. Redact it in logs, markdown, and final answers.

## Core Workflow

### 1. Resolve runtime inputs

Resolve values in this order:

Base URL:
1. explicit user-provided base URL
2. shell `OPENNOMOS_MCP_BASE_URL`
3. repo-root `.env` `OPENNOMOS_MCP_BASE_URL`
4. `https://api.opennomos.com`

Token:
1. explicit user-provided `nk_` key
2. shell `OPENNOMOS_MCP_KEY`
3. repo-root `.env` `OPENNOMOS_MCP_KEY`
4. ask the user for a key

Shell environment values override `.env`. Read `.env` directly when needed, but do not source it unless the caller explicitly wants shell state changed.

### 2. Verify connectivity

Run:

```bash
curl -sS -i "$BASE_URL/health"
```

For localhost only:

```bash
curl --noproxy '*' -sS -i "$BASE_URL/health"
```

Interpretation:

- `200` means the backend is reachable
- `502` on localhost often means the shell used a proxy instead of direct local access
- connection refused means the backend is not listening yet

### 3. Verify the credential and current actor

Use one low-cost MCP endpoint before task validation or ledger interpretation:

```bash
curl -sS -i \
  -H "Authorization: Bearer $TOKEN" \
  "$BASE_URL/api/v1/mcp/me/points"
```

For localhost only, add `--noproxy '*'`.

Interpretation:

- `200` means the token/key works
- `401` means the token/key is invalid or revoked
- `403` means the token/key is valid but lacks permission

For a `200`, keep the response as actor evidence. Do not print the token. Use returned user/account fields when present; otherwise state that the key was accepted and that subsequent `/event-stream` calls are scoped to the authenticated user.

Do not use `active_projects` or balances from `/me/points` as the only project discovery source. They show current point state, not the full set of projects worth checking.

### 4. Discover projects and task rules

For broad project discovery:

1. Call `GET /api/v1/mcp/projects` for projects accessible to the current key.
2. Call `GET /api/v1/mcp/explore/projects` for platform-level discoverable projects.
3. Normalize project IDs:
   - `/projects`: prefer `project_id`; fall back to `id` only when needed.
   - `/explore/projects`: prefer `id`; fall back to `project_id` only when needed.
4. De-duplicate project IDs before calling per-project endpoints.
5. Respect pagination (`page`, `page_size`, `limit`, `total_pages`, or an empty page) when the user asks for a complete inventory.
6. For each relevant project, call `GET /api/v1/mcp/projects/:project_id/tasks`.
7. Treat `404` from `/tasks` as "no task rule configured", not as transport failure.

When summarizing, distinguish "accessible to this key" from "discoverable on the platform".

### 5. Choose the right endpoint group

Use these patterns:

- Need to see what projects the current user can touch:
  - `GET /api/v1/mcp/projects`
- Need to see public or platform-level discoverable projects:
  - `GET /api/v1/mcp/explore/projects`
- Need detail for one project:
  - `GET /api/v1/mcp/projects/:project_id`
- Need to know what tasks are available:
  - `GET /api/v1/mcp/projects/:project_id/tasks`
- Need project KPI/overview:
  - `GET /api/v1/mcp/projects/:project_id/overview`
- Need project daily operational metrics:
  - `GET /api/v1/mcp/projects/:project_id/daily-ops`
- Need to check whether my task event was recorded:
  - `GET /api/v1/mcp/projects/:project_id/event-stream`
- Need the user's total points:
  - `GET /api/v1/mcp/me/points`
- Need points or ledger for one project:
  - `GET /api/v1/mcp/me/projects/:project_id/points`
  - `GET /api/v1/mcp/me/projects/:project_id/ledger`

Read [references/endpoints.md](references/endpoints.md) for the exact table and curl examples.

### 6. Interpret results correctly

Apply these rules:

- `404` from `/tasks` usually means the project has no configured task rule yet
- `projects` can contain both project records and application records; distinguish them before summarizing
- `/api/v1/mcp/projects/:project_id/event-stream` is the source of truth for “did my task event enter OpenNomos raw ingestion?”
- `ledger` is accounting-level data and may show types like `daily_emission` or `rollback` instead of task event names
- if the user asks “how did I get these points,” explain what the ledger confirms and clearly label any inference from task rules

### 7. Summarize for the user

Prefer:

- concrete counts
- project names
- task names and scores
- date ranges
- explicit notes on missing rules or missing detail
- evidence fields: endpoint, HTTP status, project ID, event type or ledger type, timestamp, match reason, and final outcome

Avoid:

- pasting raw JSON unless the user explicitly asks for it
- saying “MCP method” when the current route is just a REST endpoint
- exposing `nk_` keys or secrets from shell or `.env`

## Common Task Patterns

### List current platform projects

1. Call `/api/v1/mcp/projects` to get the current key's accessible projects.
2. Call `/api/v1/mcp/explore/projects` to get discoverable platform projects.
3. Normalize and de-duplicate IDs before per-project calls.
4. If the user asks for all projects, paginate until `total_pages` is exhausted or a page returns no items.
5. Present a clean list with source (`accessible`, `explore`, or both), rank/category/participant fields when available, and whether task rules exist.

### Find which tasks can be participated in

1. Call `/api/v1/mcp/explore/projects` or `/api/v1/mcp/projects`
2. Extract unique `project_id` values
3. Call `/api/v1/mcp/projects/:project_id/tasks` for each relevant project
4. Mark `404 project rule not found` as “no task rule configured”
5. Present task names, descriptions, scores, and limits

### Check whether my task event was recorded

1. Identify the project ID
2. Confirm the current actor with `/api/v1/mcp/me/points` if this has not already been done in the run
3. Call `/api/v1/mcp/projects/:project_id/event-stream`
4. Check only the current authenticated user's returned events
5. Match on the most relevant signal available:
   - `event_type`
   - `event_id`
   - task-related fields inside `payload_preview`
   - recent timestamp after the action was performed
6. Report one of these outcomes:
   - `recorded`: matching event appears in the stream
   - `pending`: action completed but no matching event is visible yet
   - `unclear`: nearby events exist but there is not enough evidence to match the task confidently
   - `failed`: no evidence exists after a reasonable wait or the platform clearly rejected the action

Keep tweet URLs, form submission responses, or product-side usage responses as intermediate evidence only. Event-stream confirmation remains the final arbiter for task execution.

### Check whether an MCP key works

1. Call `/health`
2. Call `/api/v1/mcp/me/points`
3. Optionally call `/api/v1/mcp/projects`
4. Confirm whether the key is active by reading the HTTP status and returned data

### Inspect daily project operations

1. Identify the project ID from `/api/v1/mcp/projects` or `/api/v1/mcp/explore/projects`.
2. Call `/api/v1/mcp/projects/:project_id/daily-ops?window=7d` for a short recent trend, or `window=30d` for a monthly view.
3. Interpret `registered_users` from `user_signup` events.
4. Interpret `daily_use_users` and `active_users` from `daily_use` events; currently `active_users` equals `daily_use_users`.
5. Use `signup_events` and `daily_use_events` for event volume, not unique-user counts.
6. If all usage fields are zero, say that no `daily_use` aggregate is visible for the requested window rather than assuming nobody used the product.

### Work with only default BASE_URL + `nk_` key

1. Use `https://api.opennomos.com` unless the user explicitly overrides it
2. Use the provided key, or read `OPENNOMOS_MCP_KEY`
3. Call `/api/v1/mcp/me/points`
4. Call `/api/v1/mcp/projects` or `/api/v1/mcp/explore/projects`
5. Call `/api/v1/mcp/projects/:project_id/tasks` as needed

### Explain a user's current points

1. Call `/api/v1/mcp/me/points`
2. Use `/me/points` for balances and actor context, not complete project discovery
3. Call `/api/v1/mcp/me/projects/:project_id/points` for each relevant project
4. Call `/api/v1/mcp/me/projects/:project_id/ledger`; paginate or filter with `date_from`, `date_to`, and `entry_type` when needed
5. Group ledger entries by `type`
6. Treat `daily_emission`, `rollback`, and other ledger types as accounting categories unless payload/details confidently link them to a task
7. Explain what is confirmed by the ledger, what is inferred from task rules, and what remains unknown

### Validate task execution

1. Use `/api/v1/mcp/projects/:project_id/event-stream` to answer “was my task event ingested?”
2. Use the `recorded` / `pending` / `unclear` / `failed` taxonomy
3. End the validation there unless the user explicitly asks about points or ledger

## Environment Variables

Use these names:

- `OPENNOMOS_MCP_KEY`: default auth key when the user did not provide one
- `OPENNOMOS_MCP_BASE_URL`: optional override for `https://api.opennomos.com`

Base URL resolution order:

1. explicit user-provided base URL
2. shell `OPENNOMOS_MCP_BASE_URL`
3. repo-root `.env` `OPENNOMOS_MCP_BASE_URL`
4. `https://api.opennomos.com`

Token resolution order:

1. explicit user-provided `nk_` key
2. shell `OPENNOMOS_MCP_KEY`
3. repo-root `.env` `OPENNOMOS_MCP_KEY`
4. ask the user for a key

Shell env wins over `.env`. Never write secret values into markdown memory, logs, or final answers.

## Localhost Notes

When the backend is on `localhost`, prefer:

```bash
curl --noproxy '*' ...
```

Reason: some environments export `http_proxy`, and plain `curl` may route the request through a proxy and produce a false `502`.

## Output Standard

When reporting results:

- lead with the answer
- include exact endpoint-backed facts and HTTP statuses when relevant
- use absolute dates when relevant
- separate “confirmed by API” from “inferred from rules”
- for task validation, report `recorded`, `pending`, `unclear`, or `failed`
- include the endpoint, project ID, event or ledger type, timestamp, and match reason for any claimed result
