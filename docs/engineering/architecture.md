Audience: Engineering
Status: Draft

Purpose
-------
Provide the canonical high-level architecture for Netbore: components, data flows, transport decisions, security model, and open questions. This is the source of truth for engineers designing LLDs and implementations.

Scope
-----
High-level design suitable for product managers and engineers. It intentionally omits low-level framing details (which belong in `docs/protocols/` or LLD documents) but includes enough detail to create implementation tickets and acceptance tests.

High-level overview
-------------------
Core components
- Edge gateway(s): public-facing servers that accept Viewer traffic (HTTPS/WSS) and route it into active tunnels.
- Control plane: manages authentication, tunnel lifecycle (create/list/delete), token issuance, and returns Edge candidates to Agents.
- Agent (host agent): a small, cross-platform program that opens a persistent connection to an Edge and forwards traffic to a local Host/Origin.
- Host / Origin: the local service or process being exposed by an Agent.
- Tunnel registry: datastore mapping active tunnels, assigned Edge nodes, and metadata used for routing.
- Certificate manager: automates TLS certificate issuance for public endpoints.
- Metrics & logging: observability for control/data plane health and per-tunnel metrics.

Data flow (summary)
Viewer -> Edge -> Agent -> Host/Origin
1. Agent requests a tunnel from Control plane via `POST /tunnels/create`.
2. Control plane returns an ordered list of Edge candidates, `ephemeral_token`, and optional `recreate_token`.
3. Agent establishes a persistent WSS connection to the selected Edge and performs an authenticated handshake.
4. Edge registers the tunnel in its runtime registry and routes incoming Viewer requests into the Agent connection.
5. Agent forwards the requests to the Host/Origin and relays responses back.

Transport choices
-----------------
- WSS (WebSocket over TLS): baseline transport for compatibility through NATs and firewalls; chosen for Milestone 1.
- QUIC / TCP: planned for later milestones to improve performance and support non-HTTP workloads. Agent negotiation of available transports is part of the protocol LLD.

Control & token model
---------------------
- `POST /tunnels/create`: the primary Control API for the prototype. Returns candidates and tokens.
- Tokens: `ephemeral_token` (short-lived JWT) for Edge handshake; `recreate_token` (optional) to request the same `tunnel_id` later. Registered management tokens grant sticky name control.

Security model and metadata
---------------------------
- TLS for public endpoints and WSS for Agent->Edge.
- The Edge may insert headers such as `X-Forwarded-For`, `X-Netbore-Client-IP`, and `X-Netbore-Agent-IP` to surface Viewer and Agent info to Hosts. IP masking is a configurable behavior: when enabled, Agent network addresses are not exposed to public Hosts and outward-facing logs.
- Internal logs may still retain Agent IPs for support/legal requests; access is audited and controlled.

Scaling notes
-------------
- Milestone 1: Edge uses an in-memory route table and co-located Control plane for simplicity.
- At scale: back the Tunnel registry with Redis or another shared store for cross-Edge lookups and consistent routing.

Open questions
--------------
- Certificate model for reserved subdomains: per-customer ACME vs managed certs.
- Exact token TTLs for anonymous vs registered tunnels.
- Multi-Edge registry design and eventual consistency model.

Where to go next
----------------
- Produce LLDs for protocol framing, message types, and error handling under `docs/protocols/`.
- Define an OpenAPI spec for the Control API and a small integration test harness.
