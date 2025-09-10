Audience: Engineering
Status: Draft

Netbore — Milestone 1 High-Level Design (HLD)
=============================================

Goal
----
Deliver a working proof-of-concept: a deployable Edge server and a Host Agent that together expose a local HTTP service to the public internet over a WSS tunnel. The HLD focuses on architecture, components, data flows, alternatives considered, and acceptance criteria. It deliberately avoids heavy optimization or production-scale concerns.

Purpose
-------
Provide a concise, actionable blueprint for Milestone 1: a single-edge prototype and a Host Agent that expose a local HTTP service over WSS. Keep scope limited to a working prototype and clear acceptance criteria.

Milestone 1 scope
------------------
- Expose a local HTTP service (Host/Origin) to the internet via a public hostname like `*.public.tunnel.netbore.com`.
- Transport: WSS between Agent and Edge.
- Control plane: minimal REST API for tunnel creation and token issuance.
- Two deployable artifacts: an Edge server Docker image and an Agent Docker image (runs locally or in a container).

Components
----------
- Edge server (single node for M1):
  - Public HTTPS listener for Viewer traffic.
  - WSS listener for Agent connections.
  - In-memory Tunnel registry for active tunnels.
  - HTTP reverse-proxy logic to forward Viewer requests into the correct Agent over WSS.
- Control plane (co-located with Edge for M1):
  - REST endpoints: /tunnels/create, /tunnels/delete, /tunnels/list, /auth/token.
  - Simple token generator (HMAC-signed short-lived tokens).
- Agent (Host agent):
  - CLI and config to request a tunnel and open a persistent WSS connection.
  - Forwards proxied HTTP requests to the Host/Origin on localhost and relays responses.
- Certificate manager (Edge):
  - For M1, use a self-signed cert or a managed cert for `public.tunnel.netbore.com` and its subdomains.
- Logging/metrics: minimal stdout logs and a few Prometheus-style metrics for connections and requests.

Data flow (detailed)
--------------------
1. Host operator runs Agent and requests a tunnel from the Control plane (CLI calls `/tunnels/create` with optional name).
2. Control plane returns a JSON payload with a management token, assigned Edge address, and the public hostname.
3. Agent opens a persistent WSS connection to the Edge and performs an authenticated handshake including the management token and requested hostname.
4. Edge registers the tunnel in its in-memory registry and starts routing Viewer requests for the hostname into the Agent connection.
5. Viewer sends an HTTPS request to the public hostname. Edge terminates TLS, maps the Host header to the tunnel, and forwards the request over the WSS stream to the Agent.
6. Agent translates the WSS message into an HTTP request to the local Host/Origin (e.g., localhost:8080), reads the response, and sends it back to the Edge which forwards to the Viewer.

Protocol proposal (summary)
--------------------------
- Transport: WSS, with a lightweight JSON-based control handshake and framed binary messages for request/response payloads.
- Framing model: each proxied HTTP request is assigned a stream ID. Messages for request headers, body chunks, and response parts are framed with the stream ID. Simpler than full HTTP/2 but allows basic multiplexing.

Control API: POST /tunnels/create (summary)
--------------------------------------------
This section documents the canonical `POST /tunnels/create` request and response shapes for Milestone 1. The design follows the simpler, single-create flow: no separate bind API; the Edge marks a tunnel active when the Agent successfully completes the handshake. Agents may store a `recreate_token` returned by create to re-create the same name later.

Request body (JSON)
{
  "requested_name": "string (optional)",
  "preferred_transports": ["wss","quic","tcp"],
  "region_hint": "string (optional)",
  "recreate_token_ttl_seconds": 3600, /* optional desired TTL */
  "management_token": "string (optional)"
}

Notes on request fields
- `requested_name`: optional; honored only when caller is authorized (management_token) or the name is free.
- `management_token`: may also be used as a recreate token when appropriate (i.e., a valid management token implicitly authorizes recreation/claiming of a name).
- `recreate_token_ttl_seconds`: advisory; server clamps to allowed maximum.

Response body (JSON)
```
{
  "tunnel_id": "string",
  "public_hostname": "string",
  "candidates": [
    { "edge_id":"string", "edge_url":"uri", "supported_transport":"wss|quic|tcp", "region":"string", "note":"optional" }
  ],
  "ephemeral_token": "jwt-string",
  "recreate_token": "string (optional)",
  "candidates_expires_at": "ISO-8601 timestamp",
  "note": "string (optional)"
}
```

Response details and semantics
- `candidates` is an ordered array (array order == priority). Server returns a small number (1–3). Each candidate lists a single `supported_transport` for simplicity; duplicate transports across candidates are acceptable.
- `ephemeral_token` is a short-lived signed JWT used by the Agent to authenticate the Edge handshake. Verify `exp` claim.
- `recreate_token` is an optional moderately long-lived token (signed JWT or opaque string) the agent may store locally to re-create the same tunnel/name later. The server will accept a valid `recreate_token` in subsequent `POST /tunnels/create` calls to return a fresh `ephemeral_token` and candidates for the same `tunnel_id`.
- `candidates_expires_at` is the canonical timestamp after which agents should refresh candidates.

JWT guidance
------------
- `ephemeral_token` claims (suggested): { iss, sub: <tunnel_id>, exp, iat, transports?, aud? }
- `recreate_token` may be a signed JWT `{ iss, sub: <tunnel_id>, name, exp }` (stateless) or an opaque token. For Milestone 1 prefer a signed JWT to avoid server-side token storage and revocation complexity.

Agent flow (concise)
-------------------
- First run: POST `/tunnels/create` -> receive `ephemeral_token`, `recreate_token` (optional), and `candidates`.
- Attempt to connect to candidates in array order (primary first). On successful handshake Edge marks tunnel active in registry.
- On ephemeral token expiry: if `recreate_token` exists, call POST `/tunnels/create` with `recreate_token` to request the same name and obtain a new `ephemeral_token`.
- If no `recreate_token` or it is invalid, POST `/tunnels/create` again (may receive new random name).
- Use short Edge grace windows and heartbeats to tolerate transient disconnects without re-creating.

Security and ops notes
----------------------
- Ephemeral tokens must be short-lived and validated by Edges. `recreate_token` TTL is client-choosable within policy limits; server will clamp and report the effective policy in `note`.
- No revocation is required for anonymous recreate tokens in M1; customers are responsible for secure storage. Management accounts provide account-level capabilities and long-term name reservations.

- Alternatives considered and justification
-----------------------------------------
- WebSocket (chosen) vs raw TCP: WebSocket chosen for NAT/Firewall friendliness and TLS by default.
- In-memory Edge registry vs multi-node registry: In-memory chosen for simplicity for M1. The multi-edge consistency design (how multiple Edge nodes share/update route state) is not finalized and will be addressed in follow-up designs.
- JSON-based handshake vs binary protobuf: JSON chosen for faster prototyping and easier debugging. Protobuf can be adopted later for performance and compactness.

Security considerations
----------------------
- TLS for Viewer-facing endpoints and WSS for Agent connections.
- Management tokens short-lived (default 24 hours) for M1; refresh via `/auth/token`.
- Edge will add `X-Forwarded-For` and `X-Netbore-Client-IP` headers. IP masking (masking Agent IPs from Hosts/public audiences) is out-of-scope for M1; Netbore may retain Agent IPs internally for support and lawful requests.

Operational acceptance criteria
-------------------------------
- A containerized Edge and Agent can be started locally to demonstrate a tunnel to a local Host/Origin.
- A Viewer can successfully load an HTTP page served by the Host via the public hostname.
- Control-plane APIs respond to create/delete/list operations.
- Agent reconnects after transient network failures and re-registers the tunnel.

Non-goals for M1
----------------
- Performance tuning, high availability, multi-edge consistency, production-grade certificate automation, and billing.

Next steps (after HLD approval)
-------------------------------
1. Produce the LLD for the tunnel protocol (framing, message types, error handling).
2. Define the Control API OpenAPI spec and a minimal test harness.
3. Implement quick prototypes (Edge Dockerfile and Agent Dockerfile) and a local integration test.
