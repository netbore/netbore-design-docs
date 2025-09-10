Audience: Engineering
Status: Draft

Purpose
-------
Provide a shared set of terms and definitions used across the Netbore docs to avoid ambiguity between product and engineering teams.

Scope
-----
Canonical definitions referenced by `architecture.md`, `feature-list.md`, and other docs. Keep concise and stable.

Glossary
-------
- Edge (gateway): public-facing Netbore server(s) handling Viewer traffic.
- Agent (host agent): the installed program that opens a persistent connection to an Edge and forwards to a Host/Origin.
- Host / Origin: local service/process exposed by an Agent (for example, a local web server).
- Viewer (client): external internet client (browser, webhook sender, game client) that connects to a Netbore public URL.
- Control plane: services and APIs for authentication, tunnel lifecycle, and management.
- Data plane: components forwarding traffic between Edge and Agent.
- Ephemeral token: short-lived JWT used for Edge handshake and authentication.
- Recreate token: longer-lived token (JWT or opaque) used to re-create a previously granted tunnel/name.
- Tunnel registry: datastore mapping active tunnels to runtime state and routing information.
