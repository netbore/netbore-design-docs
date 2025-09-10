Audience: Product, Engineering
Status: Draft

Purpose
-------
Compare Netbore against common alternatives. Help product managers and engineers decide when Netbore is the right fit and where trade-offs exist.

Scope
-----
High-level strengths/weaknesses versus alternatives. No performance microbenchmarks; the focus is decision guidance, not exhaustive benchmarking.

Competitors and alternatives
----------------------------
- Ngrok (hosted + enterprise)
  - Strengths: polished UX, dashboards, mature enterprise features and on-prem options.
  - Weaknesses vs Netbore: less emphasis on privacy-first defaults for hosted users; integration lock-in.
  - When to choose Ngrok: you need mature dashboards, immediate enterprise billing/SSO, and polished UX.

- Cloudflare Tunnel / Argo Tunnel
  - Strengths: global edge, TLS automation, DDoS protection, deep platform integration.
  - Weaknesses vs Netbore: binds you to Cloudflare's stack and policies; less focus on IP masking and hobbyist privacy features.
  - When to choose Cloudflare: you already use Cloudflare or need CDN/edge+DDoS protections.

- Localtunnel / expose / localhost.run
  - Strengths: extremely low friction, usually free or simple hosted projects.
  - Weaknesses vs Netbore: limited features, fewer transport options, weaker reliability, lacking in-flight encryption, and no privacy guarantees.
  - When to choose them: you want an ultra quick throwaway URL for demos or one-off testing.

- SSH reverse tunnels / self-hosted VPS designs
  - Strengths: full control, simple building blocks.
  - Weaknesses vs Netbore: operational burden (TLS, routing, certs) and no multiplexed transports or management UX.
  - When to choose DIY: you need full control over infrastructure and can manage ops costs.

- Tailscale/ZeroTier + relay nodes
  - Strengths: secure mesh networking and relays for private connectivity.
  - Weaknesses vs Netbore: not aimed at exposing public hostnames to the internet.
  - When to choose mesh: you need private connectivity between machines rather than public exposure.

Netbore positioning
-------------------
Netbore aims to sit between the ultra-simple free tunnels and the full-blown enterprise tunneling platforms. Its sweet spot is:
- Users who want simple, fast setup with privacy-forward defaults.
- Teams wanting non-HTTP support in the future (games, UDP/TCP).
- Operators who prefer a small, embeddable agent and optional self-hosting.

Decision guidance (short)
-------------------------
Choose Netbore when privacy defaults, transport flexibility, and low-friction developer UX are priorities. Defer or use alternatives if you need mature enterprise features (SSO, billing, large-scale DDoS protection) on day one.
