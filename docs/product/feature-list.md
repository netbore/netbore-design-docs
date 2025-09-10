Audience: Product, Engineering
Status: Draft

Purpose
-------
List prioritized features for the MVP and near-term roadmap, with a short statement of scope and acceptance criteria for each feature area.

Scope
-----
Covers Milestone 1 (MVP) scope and the next 2–3 milestones of incremental capability. Does not include full enterprise features (billing/SSO) which are intentionally deferred.

Core features (M1 - initial scope)
---------------------------------
- Quick-start UX: single binary/CLI and commands like `netbore tunnel http 8080` that create a public URL.
- Secure transport: WSS between Agent and Edge; HTTPS/managed certs for public endpoints.
- Control API: `POST /tunnels/create` returns candidates and tokens (ephemeral + optional recreate token).
- Agent: persistent connection to Edge, forwards requests to local Host/Origin and relays responses.
- Minimal observability: logs and basic metrics for connections and requests.

Acceptance criteria (M1)
- Edge and Agent containers can be run locally to expose a local HTTP service on a `*.public.tunnel.netbore.com` hostname.
- A Viewer (curl or browser) receives the Host's response via the public hostname.
- Control API responds to create/list/delete operations.

Near-term features (M2–M3)
-------------------------
- Accounts and sticky names: management tokens, reserved names, and account-backed recreate flow.
- Privacy controls: optional IP masking default for hobbyist users and audited internal access for support.
- Additional transports: TCP/UDP adapters and QUIC/HTTP/3 support for better performance and non-HTTP workloads.
- CNAME/reserved subdomain support and basic certificate automation.

Non-goals (initial)
-------------------
- No enterprise billing, SSO, or full multi-edge consistency for the MVP.
- No heavy per-connection content inspection or proxying at launch.

Risks and mitigations
---------------------
- Operational complexity of TLS and certificates: mitigate by using short-lived managed certs for the prototype and documenting certificate automation as a follow-up task.
- Privacy guarantees vs legal obligations: mask outward-facing data but retain internal logs under strict access controls and audit.

Next steps
----------
- Convert feature acceptance criteria into small engineering tickets (Edge, Agent, Control API, Smoke tests).
- Produce LLDs and an OpenAPI spec for `POST /tunnels/create` and related endpoints.
