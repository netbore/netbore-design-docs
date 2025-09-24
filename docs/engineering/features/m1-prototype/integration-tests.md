Audience: Engineering
Status: Draft

Integration test plan — Milestone 1
---------------------------------

Purpose
-------
Provide a minimal set of reproducible acceptance tests implementers can run locally or in CI to validate the Edge and control-plane implement the M1 protocol.

Scope
-----
Covers authentication extraction, handshake correctness, stream lifecycle, flow-control behaviour, recreate semantics, error mapping, and heartbeat/idle handling. Does not prescribe language or test framework; examples assume a small control-plane stub is available.

Test harness notes
------------------
- Tests assume a control-plane test instance (or mocked HTTP endpoints) and an Edge binary/test harness capable of speaking WSS and raw framed binary after handshake.
- Where possible prefer automated tests (Go/Python/Node.js) that spin up a control-plane stub and the Edge process.

Core test cases
---------------
1) Auth extraction matrix
   - Valid token via `Authorization: Bearer <jwt>` header — expect handshake success and accepted streams.
   - Valid token via `Sec-WebSocket-Protocol: nb_token|<jwt>` — expect handshake success.
   - Valid token via `?nb_token=<jwt>` query param — expect handshake success.
   - Precedence checks: when both header and query param present, `Authorization` header wins. When header absent but `Sec-WebSocket-Protocol` present, it wins.
   - Invalid token: expect handshake to be rejected with ERROR frame and connection close.

2) Handshake validation
   - Send minimal valid JSON handshake and expect ACCEPT response. Validate server returns expected `tunnel_id` and assigned limits.
   - Send handshake missing required fields — expect handshake reject with structured error payload.

3) Stream lifecycle and multiplexing
   - Open N concurrent streams (N = max_concurrent_streams default 32). Send interleaved REQ_HEADERS/REQ_BODY_CHUNK/REQ_END for each stream. Validate RES_* frames per stream.
   - Exceed max streams: ensure new streams are rejected with stream-level ERROR and connection remains healthy.

4) Flow-control / backpressure
   - Configure per-stream outstanding bytes = small value (e.g., 32 KiB) and stream large body. Confirm server enforces the outstanding byte limit and resumes after consumption.

5) Chunking & ordering
   - Send body in multiple REQ_BODY_CHUNK frames out of typical order and confirm server reassembles correctly and respects stream id.

6) Recreate semantics
   - Use `recreate_token` immediately after Agent reconnect. Confirm server maps new connection to previous streams when `recreate_token` is valid and within grace window.
   - Use invalid/expired recreate_token: server should reject recreate attempt and allow a fresh create sequence.

7) Error code mapping
   - Trigger known error codes (bad-auth, stream-too-large, unsupported-version) and confirm client receives ERROR frames mapped to HTTP-like codes in the payload.

8) Heartbeat & idle timeouts
   - Do not send heartbeat; confirm connection is closed after idle timeout window (heartbeat interval × max-missed).
   - Send heartbeats at interval; connection remains active.

Smoke / CI steps
----------------
- Add a small script that runs the harness tests above in a Docker container or using local binaries. Fail fast on any mismatch.
- Record minimal metrics on CI runs: handshake latency, stream throughput, error rates.

Notes
-----
- These tests are intentionally implementation-agnostic. For M1, provide a control-plane stub that verifies token signatures using a test JWKS and issues predictable limits to the Edge.

Acceptance criteria
-------------------
- All core test cases pass on a reference implementation (Edge + control-plane stub) before merging M1 changes.
