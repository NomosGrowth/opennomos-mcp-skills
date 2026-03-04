---
name: opennomos-event-troubleshooting
description: Use this skill when event quality, missing events, duplicate events, or pipeline health issues are reported for a project. Do not use it for business KPI storytelling.
---

# Opennomos Event Troubleshooting

## required_scopes
- `mcp.tools.read`

## required_tools
- `project_event_health.get`
- `project_event_stream.list`

## workflow
1. Pull event health to capture failure/duplicate rates and top error patterns.
2. Pull event stream sample (`limit=200` by default) for concrete event-level evidence.
3. Identify likely root causes grouped by schema, client behavior, and delivery pipeline.
4. Produce remediation checklist with severity and validation steps.

## output_contract
- `incident_summary`: one paragraph with impact scope.
- `root_causes`: ranked list with evidence for each.
- `remediation_plan`: ordered tasks with owner suggestions.
- `verification_checks`: explicit checks to confirm recovery.

