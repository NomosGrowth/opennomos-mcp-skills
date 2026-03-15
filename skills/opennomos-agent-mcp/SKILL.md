---
name: opennomos-agent-mcp
description: Query the OpenNomos MCP REST layer with the default base URL `https://api.opennomos.com` and an `nk_` API key. Use when tasks involve analyzing OpenNomos projects, available tasks, project overview, my task event stream, user points, or points ledger from the live backend.
---

# OpenNomos Agent MCP

Use this skill to inspect live OpenNomos MCP data through the current REST implementation.

Important: this skill is for the current OpenNomos MCP REST layer, not a standard MCP transport server. Treat it as authenticated HTTP API access.

## Quick Start

1. Default `BASE_URL` to `https://api.opennomos.com` unless the user explicitly provides another host.
2. Determine the `nk_` API key.
3. If the key is not in the user message, first check environment variable `OPENNOMOS_MCP_KEY`.
4. If no key is available in either place, stop and ask the user to provide one.
5. For localhost, call `curl --noproxy '*'` to avoid local proxy interference.
6. Check `GET /health` first when connectivity is unclear.
7. Pick the smallest set of MCP endpoints needed for the question.
8. Summarize the live result in plain language instead of dumping raw JSON.

## Inputs To Collect

- Base URL:
  - default: `https://api.opennomos.com`
  - override only when the user gives a different host such as `http://localhost:8100`
- One auth credential:
  - MCP API key starting with `nk_`

Default assumption:

- if the user only gives `BASE_URL + nk_ key`, proceed directly with MCP data routes
- if the user gives no key, check `OPENNOMOS_MCP_KEY`
- if the environment variable is empty, ask the user for a key before live calls

If the user asks for current platform state, always query the live endpoint. Do not answer from memory.

## Core Workflow

### 1. Resolve runtime inputs

Use these defaults:

- `BASE_URL=https://api.opennomos.com`
- `TOKEN` from user-provided `nk_` key

Fallback:

- if the key is missing from the request, read `OPENNOMOS_MCP_KEY`
- if `OPENNOMOS_MCP_KEY` is missing, ask the user for the key

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

### 3. Verify the credential when needed

Use one low-cost MCP endpoint first:

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

### 4. Choose the right endpoint group

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
- Need to check whether my task event was recorded:
  - `GET /api/v1/mcp/projects/:project_id/event-stream`
- Need the user's total points:
  - `GET /api/v1/mcp/me/points`
- Need points or ledger for one project:
  - `GET /api/v1/mcp/me/projects/:project_id/points`
  - `GET /api/v1/mcp/me/projects/:project_id/ledger`

Read [references/endpoints.md](references/endpoints.md) for the exact table and curl examples.

### 5. Interpret results correctly

Apply these rules:

- `404` from `/tasks` usually means the project has no configured task rule yet
- `projects` can contain both project records and application records; distinguish them before summarizing
- `/api/v1/mcp/projects/:project_id/event-stream` is the source of truth for “did my task event enter OpenNomos raw ingestion?”
- `ledger` is accounting-level data and may show types like `daily_emission` or `rollback` instead of task event names
- if the user asks “how did I get these points,” explain what the ledger confirms and clearly label any inference from task rules

### 6. Summarize for the user

Prefer:

- concrete counts
- project names
- task names and scores
- date ranges
- explicit notes on missing rules or missing detail

Avoid:

- pasting raw JSON unless the user explicitly asks for it
- saying “MCP method” when the current route is just a REST endpoint

## Common Task Patterns

### List current platform projects

1. Call `/api/v1/mcp/explore/projects`
2. If needed, call `/api/v1/mcp/projects` to compare against the current user's accessible set
3. Present the result as a clean list with rank, category, participant count, and whether tasks exist

### Find which tasks can be participated in

1. Call `/api/v1/mcp/explore/projects` or `/api/v1/mcp/projects`
2. Extract unique `project_id` values
3. Call `/api/v1/mcp/projects/:project_id/tasks` for each relevant project
4. Mark `404 project rule not found` as “no task rule configured”
5. Present task names, descriptions, scores, and limits

### Check whether my task event was recorded

1. Identify the project ID
2. Call `/api/v1/mcp/projects/:project_id/event-stream`
3. Check only the current authenticated user's returned events
4. Match on the most relevant signal available:
   - `event_type`
   - `event_id`
   - task-related fields inside `payload_preview`
   - recent timestamp after the action was performed
5. Report one of these outcomes:
   - `recorded`: matching event appears in the stream
   - `not_found`: no matching event appears in the returned window
   - `unclear`: nearby events exist but there is not enough evidence to match the task confidently

### Check whether an MCP key works

1. Call `/health`
2. Call `/api/v1/mcp/me/points`
3. Optionally call `/api/v1/mcp/projects`
4. Confirm whether the key is active by reading the HTTP status and returned data

### Work with only default BASE_URL + `nk_` key

1. Use `https://api.opennomos.com` unless the user explicitly overrides it
2. Use the provided key, or read `OPENNOMOS_MCP_KEY`
3. Call `/api/v1/mcp/me/points`
4. Call `/api/v1/mcp/projects` or `/api/v1/mcp/explore/projects`
5. Call `/api/v1/mcp/projects/:project_id/tasks` as needed

### Explain a user's current points

1. Call `/api/v1/mcp/me/points`
2. Call `/api/v1/mcp/me/projects/:project_id/points` for each active project
3. Call `/api/v1/mcp/me/projects/:project_id/ledger`
4. Group ledger entries by `type`
5. Explain what is confirmed by the ledger and what requires inference from task rules

### Validate task execution

1. Use `/api/v1/mcp/projects/:project_id/event-stream` to answer “was my task event ingested?”
2. End the validation there unless the user explicitly asks about points or ledger

## Environment Variables

Use these names:

- `OPENNOMOS_MCP_KEY`: default auth key when the user did not provide one
- `OPENNOMOS_MCP_BASE_URL`: optional override for `https://api.opennomos.com`

Resolution order:

1. explicit user-provided base URL
2. `OPENNOMOS_MCP_BASE_URL`
3. `https://api.opennomos.com`

And for the token:

1. explicit user-provided `nk_` key
2. `OPENNOMOS_MCP_KEY`
3. ask the user for a key

## Localhost Notes

When the backend is on `localhost`, prefer:

```bash
curl --noproxy '*' ...
```

Reason: some environments export `http_proxy`, and plain `curl` may route the request through a proxy and produce a false `502`.

## Output Standard

When reporting results:

- lead with the answer
- include exact endpoint-backed facts
- use absolute dates when relevant
- separate “confirmed by API” from “inferred from rules”
