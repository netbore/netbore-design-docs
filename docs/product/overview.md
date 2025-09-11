Audience: Product, Engineering
Status: Draft

Purpose
-------
Provide a concise executive summary of Netbore for product and engineering stakeholders: what the product is, who it's for, why it matters, and the measurable outcomes we expect from early prototypes.

Scope
-----
High-level product pitch, core differentiators, key success metrics, and a short recommendation for a first experiment (Milestone 1 prototype). This file intentionally omits low-level protocol details which live in [architecture.md](../engineering/architecture.md) and later LLDs.

Summary
-------
Netbore is a privacy-forward tunneling service that makes local services securely and easily reachable from the public internet. It targets developers, hobbyists, indie game hosts, and small teams who need short-lived or reserved public hostnames for local HTTP, TCP, or UDP services without complex network configuration.

Why Netbore
-----------
- Privacy-first defaults (optional IP masking) for home and hobbyist hosts.
- Transport and compatibility-first: WSS baseline with a roadmap for QUIC/TCP/UDP.
- Low-friction agent UX (single binary/CLI) with anonymous and account-backed flows.
- Operationally pragmatic: simple tokens, ordered Edge candidates, and a reproducible prototype path.

Key success metrics (examples)
- Time-to-first-tunnel: < 60s from install to reachable URL.
- Mean edge→origin latency: < 100ms in target regions for typical HTTP requests.
- 99.9% uptime for control and data planes for paid/reserved tunnels (later milestone).
- Customer-rated privacy score (survey) > 4/5 for home-hosting users.

Recommended first experiment
----------------------------
Build Milestone 1: two container images (Edge + Agent) implementing `POST /tunnels/create`, WSS-based tunnel handshake, and an end-to-end smoke test that verifies a public hostname proxies to a local Host. This delivers a tangible demo for users and validates the token/handshake model.

Next steps
----------
- Approve Milestone 1 HLD and produce LLDs for the tunnel protocol and control API (OpenAPI).
- Build the prototype and run the acceptance tests described in [roadmap.md](../roadmap.md).
