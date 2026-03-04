---
name: opennomos-growth-analysis
description: Use this skill when the user asks for growth diagnosis, KPI interpretation, or periodic performance review of a specific project. Do not use it for OAuth configuration edits.
---

# Opennomos Growth Analysis

## required_scopes
- `mcp.tools.read`

## required_tools
- `project_overview.get`
- `project_event_health.get`
- `project_top_users.list`

## workflow
1. Read project overview with `window=30d` to establish baseline KPI trend.
2. Read event health to identify delivery quality constraints.
3. Read top users to detect concentration risk and user mix skew.
4. Build a concise diagnosis with three sections: summary, evidence, actions.

## output_contract
- `summary`: one short paragraph, plain language.
- `evidence`: bullet list of concrete metrics from tool outputs.
- `actions`: 3-5 prioritized actions with expected impact.
- `confidence`: high/medium/low based on data completeness.

