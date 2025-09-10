Audience: Product, Engineering
Status: Draft

Purpose
-------
Describe the target users, personas, and high-value use cases to align product and engineering priorities.

Scope
-----
Personas, primary use cases, and what success looks like for each persona. Does not prescribe exact pricing or commercial packaging.

Personas
--------
- Individual developers
  - Needs: quick public URLs for local testing, webhooks, or demos.
  - Success: able to create an anonymous tunnel and share a URL in under 60s.

- Small teams / contractors
  - Needs: temporary exposure for demos or client previews, some persistent names for repeated demos.
  - Success: sticky names for short project lifetimes and an account-backed flow for reserved subdomains.

- Indie game devs / modders / local server operators
  - Needs: expose game servers or UDP/TCP services with low friction.
  - Success: reliable connectivity and a roadmap to non-HTTP transports.

- Privacy-conscious home hosts
  - Needs: public exposure without revealing home IPs to public audiences.
  - Success: IP masking by default and transparent operational controls for support/legal access.

High-value use cases
--------------------
- Webhook development: receive events from external services to a local dev machine.
- Live demos and client previews: share a URL to a running feature without deploying.
- Home-hosted services: allow others to connect to a self-hosted NAS, LLM, or game server.
- Embeddable client integrations: library or SDK for third-party apps to embed tunneling features.

Success signals
---------------
- High conversion from anonymous tunnels to registered accounts when users need sticky names.
- Low time-to-first-tunnel metrics and positive privacy feedback from home-hosting users.
