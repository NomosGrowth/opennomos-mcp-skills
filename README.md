# OpenNomos MCP Skills

Reusable Codex skills for operating OpenNomos MCP.

## Included Skills

- `opennomos-agent-mcp`

## Install

```bash
git clone https://github.com/<your-org>/opennomos-mcp-skills.git
mkdir -p "$CODEX_HOME/skills"
cp -R opennomos-mcp-skills/skills/* "$CODEX_HOME/skills/"
```

## Update

```bash
cd opennomos-mcp-skills
git pull
cp -R skills/* "$CODEX_HOME/skills/"
```

## Validate (optional)

```bash
python3 "$CODEX_HOME/skills/.system/skill-creator/scripts/quick_validate.py" \
  "$CODEX_HOME/skills/opennomos-agent-mcp"
```
