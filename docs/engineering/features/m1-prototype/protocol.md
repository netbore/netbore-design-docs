Audience: Engineering
Status: Draft

Netbore — Milestone 1 Protocol Specification
=================================================

This file documents the WSS handshake and the binary frame protocol used between Edge and Agent for M1.

1) Connection establishment (handshake)
- Transport: WSS (WebSocket over TLS). The Agent opens a WebSocket and MUST send a single text-frame immediately containing a JSON handshake message. Edge responds with a text-frame JSON handshake response.

Agent -> Edge handshake JSON (text frame)
{
  "type": "handshake",
  "ephemeral_token": "<jwt>",
  "requested_hostname": "myapp.public.tunnel.netbore.com",
  "agent_id": "optional-agent-id",
  "agent_version": "0.1.0",
  "meta": { "local_addr": "127.0.0.1:8080" }
}

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
- Edge validates ephemeral_token: signature, expiry (`exp`), and optional `sub`==tunnel_id.
- If valid, Edge reserves the requested_hostname (if available) and returns status: ok. If requested name conflicts, Edge may return status: error with note.

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