---
name: opennomos-project-inspection
description: Use this skill to inspect project OAuth client setup and, when authorized, update redirect URI configuration. Do not use it without explicit project context.
---

# Opennomos Project Inspection

## required_scopes
- `mcp.tools.read`
- `mcp.tools.write` (only when applying redirect URI updates)

## required_tools
- `oauth_client.get`
- `oauth_redirect_uris.update`

## workflow
1. Fetch current OAuth client configuration for the target project.
2. Evaluate redirect URI quality against security and environment rules.
3. If user asks for fix and write scope exists, apply URI update with explicit list.
4. Re-fetch config to verify final state and report differences.

## output_contract
- `current_state`: existing client and URI summary.
- `issues`: list of config risks or mismatches.
- `changes_applied`: null if read-only, otherwise exact URI diff.
- `next_steps`: short operational guidance.

