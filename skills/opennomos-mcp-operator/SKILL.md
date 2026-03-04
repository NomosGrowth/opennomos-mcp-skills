---
name: opennomos-mcp-operator
description: "Use this skill to operate and troubleshoot OpenNomos MCP integration end-to-end: verify OAuth and protected-resource metadata, obtain access tokens (prefer client_credentials for machine clients), discover tools, execute tool calls, and diagnose auth or scope failures. Trigger when users ask how to connect AI/Codex to OpenNomos MCP, how to call MCP tools, or how to query their own projects, points, and project detail/rule data via MCP."
---

# OpenNomos MCP Operator

## Workflow

1. Verify discovery endpoints first.
- Read `https://api.opennomos.com/.well-known/oauth-authorization-server`.
- Read `https://opennomos.com/.well-known/oauth-protected-resource`.
- Confirm `issuer`, `token_endpoint`, `jwks_uri`, and protected `resource`.

2. Obtain an access token for MCP calls.
- Prefer `grant_type=client_credentials` with `scope=mcp.tools.read` (or `mcp.tools.write` when changing redirect URIs).
- Send token request to `https://api.opennomos.com/oauth/token`.
- Keep audience/resource aligned with MCP endpoint (`https://opennomos.com/mcp`).

3. Initialize MCP session and discover tools.
- Call MCP `initialize` once.
- Call `tools/list` and verify required tools exist before `tools/call`.
- Parse `structuredContent` as primary result payload.

4. Execute user task with the minimal tool set.
- For “我的项目”: call `my_projects.list`.
- For “我的分数”: call `my_points.get`.
- For “项目详情/规则”: call `project_profile.get` with `project_id`.
- For analytics or troubleshooting, use project dashboard tools only when needed.

5. Return concise, evidence-backed output.
- Include request inputs used (`project_id`, window, limit, page).
- Surface backend errors verbatim when they explain auth/scope issues.
- Suggest next action only when blocked.

## Tool Mapping

- `my_projects.list`: list projects owned/managed by current principal.
- `my_points.get`: get current principal growth points overview.
- `project_profile.get`: fetch project detail and active rule in one call.
- `projects.list`: list ranking projects (`new` or `trending`) for discovery.
- `project_overview.get`: KPI and trend summary.
- `project_event_health.get`: event quality and error summary.
- `project_event_stream.list`: raw event sample for debugging.
- `project_top_users.list`: top users and concentration analysis.
- `oauth_client.get`: inspect OAuth client for a project.
- `oauth_redirect_uris.update`: update redirect URIs (requires write scope).

## Guardrails

- Require `project_id` when project-specific tools are used.
- Avoid write tools unless user explicitly asks for a change.
- Reject ambiguous “改一下配置” requests unless exact URI list is provided.
- Keep read/write scope separation explicit in responses.

## Failure Diagnostics

- `401 invalid_token`:
- Check token signature/issuer/audience expiry; re-issue token.
- `403 insufficient_scope`:
- Request broader scope (`mcp.tools.write` for update operations).
- `403 machine token is not bound to any user principal`:
- Explain that “我的项目/我的分数” endpoints need a user principal.
- For this backend design, bind by completing one successful `authorization_code` flow for the same `client_id`, then retry `client_credentials`.
- Tool not found:
- Re-run `tools/list`; verify deployed frontend MCP version.

## Output Contract

- `task`: user intent in one line.
- `auth_state`: token scope/audience/principal-binding status.
- `tool_calls`: ordered list of MCP calls with key arguments.
- `result`: normalized payload summary from `structuredContent`.
- `errors`: empty array or concrete error diagnostics.
- `next_step`: nullable, one actionable step when blocked.

## References

- Read [references/opennomos-mcp-templates.md](references/opennomos-mcp-templates.md) for ready-to-use curl and JSON-RPC payload templates.
