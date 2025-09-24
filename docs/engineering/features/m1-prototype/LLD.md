Audience: Engineering
Status: Draft

Netbore — Milestone 1 Low-Level Design (LLD)
=================================================

This document translates the HLD into concrete protocol framing, message types, timeouts, and component contracts needed to implement the Edge and Agent for M1.

Checklist (requirements from HLD and this task)
- Provide a complete LLD describing protocol framing, message types, and stream lifecycle. [Done]
- Provide data schemas for control API request/response and tokens. [Done]
- Provide protocol specification for WSS tunnel messages and handshake. [Done]
- Surface follow-up questions if any details are underspecified. [Partially — see questions section]

Contract (tiny)
- Inputs: Viewer HTTPS requests arriving at Edge; Agent-origin HTTP requests to localhost from Agent; Control-plane REST requests to create/list/delete tunnels.
- Outputs: HTTP responses delivered to Viewer; tunnel state changes in Edge registry; tokens returned by control-plane.
- Error modes: network disconnects, invalid tokens, stream limits, oversized payloads.

Edge / Agent runtime parameters (defaults)
- Max concurrent streams per tunnel: 32
- Max header size (serialized JSON): 64KiB
- Max body chunk size per frame: 64KiB
- Heartbeat interval: 15s (agent -> edge), heartbeat timeout: 45s
- Agent reconnect backoff: jittered exponential starting at 1s, cap 30s

Stream & connection lifecycle
- Agent opens a WSS connection to an Edge candidate URL with a one-shot JSON handshake (see [protocol.md](protocol.md)).
- On successful handshake Edge transitions tunnel state -> ACTIVE and registers (in-memory) the mapping hostname -> connection.
- When a Viewer request arrives, Edge creates a new stream ID (u64) and sends REQ_HEADERS for that stream to Agent. Agent responds with RES_HEADERS/RES_BODY frames on same stream ID.
- If the Agent or Edge closes the WSS connection, Edge marks the tunnel as DOWN and clients receive 502/504 depending on timing. Agent reconnect attempts will re-activate the tunnel when handshake completes within recreate grace window.

Framing and wire-level decisions (summary)
- Handshake: JSON over first WS text message (human readable). Fields: ephemeral_token, requested_hostname, agent_info.
- Framed messages: binary frames prefixed with 4-byte big-endian length then the frame payload. Payload begins with 1-byte frame type and 8-byte stream id (big-endian uint64). Remaining bytes are frame-specific payload.
- Rationale: fixed-length stream id and type keep parsing simple in production C/Go implementations. Length prefix lets WS binary frames contain multiple logical frames if desired.

Message types (codes)
- 0x01 REQ_HEADERS — payload: JSON headers object (method, path, headers map, http_version)
- 0x02 REQ_BODY_CHUNK — payload: raw bytes
- 0x03 REQ_END — no payload
- 0x11 RES_HEADERS — payload: JSON response head (status, headers map)
- 0x12 RES_BODY_CHUNK — payload: raw bytes
- 0x13 RES_END — no payload
- 0x20 OPEN_STREAM (reserved — not used; headers start the stream)
- 0x30 HEARTBEAT — payload: optional JSON {ts}
- 0x40 ERROR — payload: JSON {code, message}
- 0x50 PING/PONG — optional for liveness; use HEARTBEAT instead for M1

Frame encoding (detailed)
- Frame layout on the wire (binary WebSocket message(s)):
  - 4 bytes: total frame length N (big-endian uint32) not including these 4 bytes
  - 1 byte: frame type (uint8)
  - 8 bytes: stream id (uint64 big-endian). For HEARTBEAT and connection-level frames, stream id MUST be 0.
  - N-9 bytes: payload (optional)

JWT claims: single source of truth
- Detailed claim shapes and JSON Schema for `ephemeral_token`, `recreate_token`, and `management_token` are maintained in [data-schemas.md](data-schemas.md). The LLD references that file as the canonical location for token claim shapes and examples.

Security notes and the threat model are documented in [threat-model.md](./threat-model.md).

Acceptance tests: see [integration-tests.md](./integration-tests.md) for the minimal acceptance test plan implementers should run against Edge and the control-plane stub.
  participant Viewer

  CP->>Agent: POST /tunnels/create
  CP-->>Agent: ephemeral_token

  Agent->>Edge: WSS Upgrade (auth via header/proto/query)
  Agent->>Edge: handshake (JSON)
  Edge-->>Agent: handshake OK

  Viewer->>Edge: HTTPS request (Host -> tunnel)
  Edge->>Agent: REQ_HEADERS (stream=1)
  Agent-->>Edge: RES_HEADERS (stream=1)
  Agent-->>Edge: RES_BODY_CHUNK(...)
  Agent-->>Edge: RES_END (stream=1)
  Edge->>Viewer: HTTP response
```

Simple data-flow diagram (Mermaid)

```mermaid
flowchart LR
  Viewer-->Edge[Edge (HTTPS terminate)]
  Edge-->WSS[WSS (binary frames) \n map Host -> tunnel]
  WSS-->Agent[Agent]
  Agent-->Origin[Origin (localhost)]
```


Operational notes
- Logs: both Agent and Edge must log at INFO connection events (connect/disconnect), and DEBUG for stream lifecycle if env DEBUG=true.
- Metrics: counters for connections, active_tunnels, streams_open, streams_closed, stream_errors.

Examples & sequence
- See [protocol.md](protocol.md) for handshake JSON and an end-to-end sequence diagram for create -> agent connect -> viewer request.

Follow-up questions (minimal)
1. For M1, should the Agent perform HTTP->HTTPS outbound to origin when Host requests HTTPS? (Assume no: origin is local HTTP; TLS termination only at Edge.)
2. Any preferred JWT claim names beyond iss/sub/exp/iat? (We included transports and aud as optional.)

Next steps
- Review this LLD. If approved, use this spec to implement the Edge/Agent prototypes and run the integration test harness defined in [HLD.md](HLD.md).