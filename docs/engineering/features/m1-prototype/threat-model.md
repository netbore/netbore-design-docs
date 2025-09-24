Audience: Engineering
Status: Draft

# Threat Model (Milestone 1)

Purpose
-------
Document attacker models, primary assets, trust boundaries, and mitigations relevant to M1. This guides implementation risk decisions and test coverage.

Scope
-----
Covers threats that arise from the Edge↔Agent protocol, JWT handling, and control-plane key distribution. Host compromise and control-plane compromise are noted but out-of-scope for M1.

## Overview
This section documents the threat model for the M1 protocol, focusing on attacker goals, trust boundaries, protocol/crypto assumptions, and mitigations. It is intended to guide implementers and reviewers in understanding the security posture and key risks of the design.

## Assets to Protect
- Viewer HTTP requests and responses (confidentiality, integrity)
- Agent credentials (ephemeral/recreate/management JWTs)
- Tunnel state and mapping (hostname → Agent)
- Control-plane API keys and signing keys
- Edge and Agent process integrity

## Trust Boundaries
- All traffic between Agent and Edge is over WSS (WebSocket over TLS); confidentiality and integrity are provided by TLS.
- JWTs are bearer tokens: possession grants access. Ephemeral tokens are short-lived; recreate/management tokens may be longer-lived.
- Edge and Agent are assumed to be mutually untrusted except for the possession of valid tokens.
- Control-plane is trusted to issue tokens and manage key rotation.

## Attacker Goals
- Steal or replay tokens to hijack tunnels or impersonate Agents
- Intercept or tamper with Viewer traffic (MITM)
- Deny service to legitimate Agents or Viewers (DoS)
- Exploit protocol parsing or implementation bugs (buffer overflow, injection)
- Downgrade protocol or bypass authentication

## Threats & Mitigations

### 1. Token Theft (Bearer Token Risk)
- **Threat:** Attacker obtains a valid JWT (ephemeral, recreate, or management) and uses it to open or hijack a tunnel.
- **Mitigations:**
  - All tokens are transmitted only over WSS (TLS).
  - Ephemeral tokens are short-lived (minutes); recreate tokens have limited TTL and are single-use.
  - JWTs are signed (HS256/RS256) and validated for `exp`, `nbf`, and `iat` with clock skew tolerance.
  - Management tokens are never sent over the tunnel protocol.
  - See [data-schemas.md](data-schemas.md) for claim details and token structure.

### 2. Token Replay
- **Threat:** Attacker replays a previously valid token to re-establish a tunnel or hijack a session.
- **Mitigations:**
  - Ephemeral tokens are single-use and expire quickly.
  - Recreate tokens are bound to a specific tunnel and have a short grace window.
  - Edge tracks active tunnel IDs and rejects duplicate or concurrent handshakes for the same hostname.

### 3. MITM (Man-in-the-Middle)
- **Threat:** Attacker intercepts or tampers with traffic between Agent and Edge.
- **Mitigations:**
  - All traffic is over WSS (TLS); certificate validation is required.
  - JWTs are validated for signature and expiry.
  - No sensitive data is sent in cleartext.

### 4. Protocol Downgrade / Bypass
- **Threat:** Attacker attempts to force a downgrade to an insecure protocol or bypass authentication.
- **Mitigations:**
  - Only WSS is supported; plain WS is rejected.
  - All handshakes require a valid JWT; no unauthenticated mode exists.
  - Token extraction precedence is strictly defined; ambiguous or multiple tokens are rejected.

### 5. Denial of Service (DoS)
- **Threat:** Attacker floods Edge or Agent with handshake attempts, streams, or large payloads.
- **Mitigations:**
  - Max concurrent streams, header/body size, and outstanding bytes are limited (see LLD).
  - Handshake and stream timeouts are enforced.
  - Edge may rate-limit or temporarily ban abusive clients.

### 6. Key/Algorithm Downgrade
- **Threat:** Attacker attempts to force use of a weak signing algorithm or stale key.
- **Mitigations:**
  - Control-plane advertises allowed algorithms; Edge enforces policy (HS256/RS256 only).
  - JWKS is fetched securely and refreshed on error or expiry.
  - Edge rejects tokens with unknown or weak `alg`.

### 7. Implementation Bugs
- **Threat:** Buffer overflows, injection, or logic errors in frame parsing or JWT validation.
- **Mitigations:**
  - Frame and handshake parsing is strict and length-prefixed.
  - All JSON is parsed with strict schema validation.
  - JWT libraries are used for signature and claim validation.
  - Fuzzing and integration tests are recommended (see [integration-tests.md](integration-tests.md)).

## Out-of-Scope (M1)
- Agent/Edge host compromise (malware, root access)
- Control-plane compromise (key theft, privilege escalation)
- Viewer-to-Edge HTTPS security (assumed to be handled by standard HTTPS stack)

## Open Questions / Future Work
- Mutual authentication (Agent authenticates Edge)
- Token binding to client IP or device
- Fine-grained scope/role claims for management tokens
- Per-stream encryption or compression
