Audience: Product, Engineering
Status: Draft

Purpose
-------
Capture the milestone roadmap for Netbore with clear scope and acceptance criteria for each milestone to guide implementation and validation.

Scope
-----
High-level milestones (M1–M5) and the acceptance tests for Milestone 1. This document is an action-oriented roadmap for PMs and engineers.

Milestones
----------
- Milestone 1 — Edge + Agent prototype (MVP)
  - Scope: WSS-based tunnels, TLS for public endpoints, `POST /tunnels/create` control API, two Docker images (Edge + Agent), in-memory tunnel registry.
  - Acceptance criteria:
    - Edge and Agent containers can be deployed locally.
    - A Viewer can `curl https://<public-hostname>/` and receive the Host's response.
    - Control API supports create/list/delete.
    - Agent reconnects after transient network drop and resumes routing.

- Milestone 2 — Accounts & sticky names
  - Scope: account model, management tokens, recreate tokens for sticky names, reserved subdomain flow.
  - Outcomes: users can register accounts, reserve names, and use persistent `recreate_token` behavior.

- Milestone 3 — Protocols & integrations
  - Scope: add TCP/UDP adapters, QUIC support, CNAME support for custom domains, embeddable clients.
  - Outcomes: non-HTTP workloads supported and improved performance.

- Milestone 4 — UX & operations
  - Scope: web UI, metrics dashboards, improved observability, reserved subdomain UI.
  - Outcomes: better onboarding and production readiness for paying customers.

- Milestone 5 — Enterprise features
  - Scope: billing, SSO, ACLs, enterprise admin tooling (deferred until product-market fit).

Milestone 1 next steps (actionables)
-----------------------------------
1. Approve HLD and produce LLDs for framing and control API.
2. Implement Edge and Agent Docker images and a minimal certificate for `public.tunnel.netbore.com` (self-signed or managed cert for prototype).
3. Implement `POST /tunnels/create` per the HLD and wire ephemeral/recreate tokens.
4. Build the smoke tests and local integration harness described in acceptance criteria.

Notes
-----
- Non-functional requirements (scalability, HA, global edge) are deliberately out-of-scope for M1 and scheduled for later milestones.
