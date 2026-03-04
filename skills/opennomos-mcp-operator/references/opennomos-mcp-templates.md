# OpenNomos MCP Templates

## Endpoints

- OAuth metadata: `https://api.opennomos.com/.well-known/oauth-authorization-server`
- JWKS: `https://api.opennomos.com/.well-known/jwks.json`
- Token endpoint: `https://api.opennomos.com/oauth/token`
- Protected resource metadata: `https://opennomos.com/.well-known/oauth-protected-resource`
- MCP endpoint: `https://opennomos.com/mcp`

## Get Client-Credentials Token

```bash
curl -sS https://api.opennomos.com/oauth/token \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=client_credentials' \
  --data-urlencode 'client_id=YOUR_CLIENT_ID' \
  --data-urlencode 'client_secret=YOUR_CLIENT_SECRET' \
  --data-urlencode 'scope=mcp.tools.read' \
  --data-urlencode 'resource=https://opennomos.com/mcp'
```

## Initialize MCP

```bash
curl -sS https://opennomos.com/mcp \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "jsonrpc":"2.0",
    "id":1,
    "method":"initialize",
    "params":{"protocolVersion":"2025-03-26","capabilities":{}}
  }'
```

## List Tools

```bash
curl -sS https://opennomos.com/mcp \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'
```

## Call My Projects

```bash
curl -sS https://opennomos.com/mcp \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "jsonrpc":"2.0",
    "id":3,
    "method":"tools/call",
    "params":{
      "name":"my_projects.list",
      "arguments":{"page":1,"page_size":20}
    }
  }'
```

## Call My Points

```bash
curl -sS https://opennomos.com/mcp \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "jsonrpc":"2.0",
    "id":4,
    "method":"tools/call",
    "params":{
      "name":"my_points.get",
      "arguments":{}
    }
  }'
```

## Call Project Profile

```bash
curl -sS https://opennomos.com/mcp \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "jsonrpc":"2.0",
    "id":5,
    "method":"tools/call",
    "params":{
      "name":"project_profile.get",
      "arguments":{"project_id":"YOUR_PROJECT_ID"}
    }
  }'
```

## Common Error Hints

- `invalid_token`: token expired, wrong issuer, wrong audience, or bad signature.
- `insufficient_scope`: missing `mcp.tools.read` or `mcp.tools.write`.
- `machine token is not bound to any user principal`: complete one user authorization flow for the same `client_id`, then retry.
