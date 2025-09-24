Audience: Engineering
Status: Draft

Netbore — Milestone 1 Protocol Specification
=================================================

This file documents the WSS handshake and the binary frame protocol used between Edge and Agent for M1.

1) Connection establishment (handshake)
- Transport: WSS (WebSocket over TLS). The Agent opens a WebSocket and MUST send a single text-frame immediately containing a JSON handshake message. Edge responds with a text-frame JSON handshake response.

Agent -> Edge handshake JSON (text frame)

Schema (JSON Schema snippet)
```
{
  "$id": "https://netbore.example/schemas/handshake.json",
  "type": "object",
  "properties": {
    "type": { "const": "handshake" },
    "ephemeral_token": { "type": "string" },
    "requested_hostname": { "type": "string" },
    "agent_id": { "type": "string" },
    "agent_version": { "type": "string" },
    "meta": { "type": "object" }
  },
  "required": ["type","ephemeral_token","requested_hostname"],
  "additionalProperties": false
}
```

Example JSON
```
{
  "type": "handshake",
  "ephemeral_token": "<jwt>",
  "requested_hostname": "myapp.public.tunnel.netbore.com",
  "agent_id": "agent-1234",
  "agent_version": "0.1.0",
  "meta": { "local_addr": "127.0.0.1:8080" }
}
```

Edge -> Agent handshake response JSON (text frame)
{
  "type": "handshake_response",
  "status": "ok|error",
  "tunnel_id": "<id>",
  "note": "optional human message",
  "server_time": "ISO-8601",
  "grace_seconds": 30
}

Handshake semantics
- Edge MUST extract the token using the WS auth precedence (see below) and then validate the token signature and `exp`/`nbf`/`iat` claims using the configured key material.
- On successful validation, Edge compares `sub` or `name` claims to `requested_hostname` if present per control-plane policy. If the name is already reserved by another active tunnel, Edge returns a handshake_response with status `error` and a human `note` describing the conflict.

Token extraction precedence (Edge MUST apply in this order):
1. `Authorization: Bearer <jwt>` header (highest precedence)
2. `Sec-WebSocket-Protocol: nb_token|<jwt>` — a token injected in the Sec-WebSocket-Protocol header using `|` as a separator
3. `?nb_token=<jwt>` query parameter (lowest precedence)

Sec-WebSocket-Protocol parsing example

Clients that cannot set `Authorization` may supply a token using the `Sec-WebSocket-Protocol` header. For M1 we prefer the format `nb_token|<jwt>` where the pipe (`|`) separates the token kind from the compact JWT. The Edge SHOULD parse the header following standard header rules (comma-separated values), locate the first entry that begins with `nb_token` (optionally followed by a separator), and take the token portion for validation.

Example header:

Sec-WebSocket-Protocol: nb_token|eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWUsImlhdCI6MTUxNjIzOTAyMn0.KMUFsIDTnFmyG3nMiGM6H9FNFUROf3wh7SmqJp-QV30

If multiple `nb_token` entries appear, Edge MUST take the first occurrence.

Versioning and separator rationale

Using a pipe (`|`) or similar safe separator makes it trivial to extend the left-hand side for versioning or metadata without changing the parsing of the JWT. Example future form:

Sec-WebSocket-Protocol: nb_token.v1|<jwt>

Edge parsers should split the token-kind and the jwt on the first separator and use the jwt part for validation while optionally using the left-hand metadata (e.g., `v1`) to adjust behaviour.

Allowed separator notes

HTTP token characters are limited; many punctuation characters are disallowed as separators. The pipe (`|`, 0x7C) is permitted and commonly available in user agents and servers and is therefore a reasonable default for this project.

Recreate / re-activate semantics
- If Agent uses a valid `recreate_token` (in lieu of ephemeral_token) during the handshake, Edge MAY accept it if the token maps to an existing tunnel_id and is within its TTL. If accepted, Edge SHOULD return `tunnel_id` in the handshake response and attempt to rebind the same `requested_hostname` to the new connection (subject to conflict rules).
- If two Agents attempt to re-activate the same hostname simultaneously, Edge must consult control-plane policy: M1 behaviour is to accept the first completed handshake and reject subsequent concurrent handshakes for the same hostname with response `error`.

JWT claim expectations
- Claim shapes and examples are documented in [data-schemas.md](data-schemas.md). Protocol implementations MUST validate token signature and `exp`, and SHOULD correlate `sub` to the requested tunnel when present.

2) Binary framed message format (after handshake)
- All subsequent frames MUST be binary WebSocket frames. A single WS message MAY contain one or more concatenated frames.

Frame binary layout (per-frame)
- 4 bytes | uint32 BE | frame_length (L) = number of bytes following these 4 bytes
- 1 byte  | uint8      | frame_type
- 8 bytes | uint64 BE  | stream_id (0 == connection-level)
- (L-9) bytes | payload bytes | frame-specific payload

Frame type codes and payloads
- 0x01 REQ_HEADERS — payload: UTF-8 JSON object {
    method: string,
    path: string,
    headers: { [string]: string | string[] },
    http_version?: string
  }
- 0x02 REQ_BODY_CHUNK — payload: raw bytes
- 0x03 REQ_END — payload: empty
- 0x11 RES_HEADERS — payload: UTF-8 JSON { status: int, headers: { ... } }
- 0x12 RES_BODY_CHUNK — payload: raw bytes
- 0x13 RES_END — payload: empty
- 0x30 HEARTBEAT — payload: optional JSON { ts: "ISO-8601" }
- 0x40 ERROR — payload: UTF-8 JSON { code: string, message: string }

Stream ID allocation
- Edge creates stream IDs for Viewer-initiated requests. IDs are uint64 values incremented per-connection starting at 1.
- Agent MUST respect stream IDs and must not create stream IDs >0. (Agents will reply on the same stream IDs they receive.)

Concurrency
- Agents should support multiplexing several streams over the same WSS connection, up to the configured max_concurrent_streams.

Time-bound behaviour
- If a stream is idle with no frames for `stream_idle_timeout` (default 60s), either side may send ERROR and close stream.

Example wire trace (pseudocode)
1) Agent sends handshake text frame with ephemeral_token
2) Edge replies handshake_response OK
3) Viewer requests GET / -> Edge creates stream 1 and sends REQ_HEADERS (type 0x01, stream 1)
4) Edge sends REQ_HEADERS frame to Agent over WSS
5) Agent performs HTTP GET to localhost, reads response headers -> sends RES_HEADERS (type 0x11)
6) Agent streams response body in chunks using RES_BODY_CHUNK and finishes with RES_END
7) Edge responds to Viewer

Notes & limitations for M1
- No per-stream flow control; for larger payloads, sending side should chunk into 64KiB payloads. Implementing windowed flow-control is a follow-up.
- No built-in compression. Implement later if needed.

Security notes
- Ephemeral JWTs must be validated and have short expiries. On token expiry mid-connection, Edge should allow the existing WSS connection to continue until closed, but refuse new handshakes with expired tokens.

For M1 prefer HMAC-SHA256 signed tokens and keep `ephemeral_token` TTL short (minutes). `recreate_token` and `management_token` have longer TTLs and include `sub` and `scope`/`role` claims as appropriate.

Examples (hex frame)
- Suppose JSON "{\"status\":200,\"headers\":{}}" is 30 bytes; frame_length = 1+8+30=39 -> 0x00 0x00 0x00 0x27, type 0x11, streamid 0x00..01, payload...

Reference: See [LLD.md](LLD.md) for implementation defaults and limits.