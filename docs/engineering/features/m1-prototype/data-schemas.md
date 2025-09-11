Audience: Engineering
Status: Draft

Netbore — Milestone 1 Data Schemas
==================================

This file contains JSON Schema fragments and example objects used by the Control-plane and by runtime components.

1) POST /tunnels/create request schema (partial)

JSON Schema (simplified)
```
{
  "$id": "https://netbore.example/schemas/tunnels-create.json",
  "type": "object",
  "properties": {
    "requested_name": { "type": "string", "maxLength": 64 },
    "region_hint": { "type": "string" },
    "recreate_token_ttl_seconds": { "type": "integer", "minimum": 60 },
    "management_token": { "type": "string" }
  },
  "required": []
}
```

Example request
```
{
  "requested_name": "myapp",
  "recreate_token_ttl_seconds": 3600
}
```

2) POST /tunnels/create response schema (simplified)

```
{
  "$id": "https://netbore.example/schemas/tunnels-create-response.json",
  "type": "object",
  "properties": {
    "tunnel_id": { "type": "string" },
    "public_hostname": { "type": "string" },
    "candidates": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "edge_id": { "type": "string" },
          "edge_url": { "type": "string", "format": "uri" },
          "supported_transport": { "type": "string" },
          "region": { "type": "string" }
        },
        "required": ["edge_id","edge_url","supported_transport"]
      }
    },
    "ephemeral_token": { "type": "string" },
    "recreate_token": { "type": "string" },
    "candidates_expires_at": { "type": "string", "format": "date-time" }
  },
  "required": ["tunnel_id","public_hostname","candidates","ephemeral_token","candidates_expires_at"]
}
```

3) JWT claim schemas (recommended)

ephemeral_token claims (JSON Schema snippet)
```
{
  "type": "object",
  "properties": {
    "iss": { "type": "string" },
    "sub": { "type": "string" },
    "iat": { "type": "integer" },
    "exp": { "type": "integer" },
    "aud": { "type": "string" },
    "name": { "type": "string" }
  },
  "required": ["iss","sub","iat","exp"]
}
```

recreate_token claims (JSON Schema snippet)
```
{
  "type": "object",
  "properties": {
    "iss": { "type": "string" },
    "sub": { "type": "string" },
    "name": { "type": "string" },
    "iat": { "type": "integer" },
    "exp": { "type": "integer" }
  },
  "required": ["iss","sub","iat","exp"]
}
```

management_token claims (JSON Schema snippet)
```
{
  "type": "object",
  "properties": {
    "iss": { "type": "string" },
    "sub": { "type": "string" },
    "iat": { "type": "integer" },
    "exp": { "type": "integer" },
    "scope": { "type": ["string","array"] },
    "role": { "type": "string" }
  },
  "required": ["iss","sub","iat","exp"]
}
```

5) Agent config file (agent.yaml) — minimal example
```
agent_id: "agent-1234"
origin: "http://127.0.0.1:8080"
control_url: "https://control.public.tunnel.netbore.com"
requested_name: "myapp"
```

6) Tunnel registry object (in-memory)
```
{
  "tunnel_id": "<id>",
  "public_hostname": "<string>",
  "edge_id": "<id>",
  "agent_conn": "<runtime-ref>",
  "state": "ACTIVE|PENDING|DOWN",
  "created_at": "ISO-8601",
  "expires_at": "ISO-8601 (candidates_expires_at)"
}
```

Quality note: these schemas are intentionally small for M1. We can expand into full JSON Schema docs and place them under `schemas/` if you want formal validation in CI.