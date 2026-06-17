# bill.today WebSocket — 120s reconnect: root cause + fix

This branch contains **only** the investigation deliverables for the
"`wss://bill.today/` reconnects every ~2 minutes" issue. (It deliberately does
not carry the opencode source tree.)

## Files
- **`bill-today-websocket-fix-prompt.md`** — full root-cause writeup and a
  self-contained fix prompt for `@push.rocks/smartproxy`: empirical evidence,
  the confirmed mechanism, a commit-history assessment (nothing to revert), the
  concrete patch + call-site wiring, and an acceptance test plan.
- **`smartproxy-connection-pool-ws-age-exemption.patch`** — the core code
  change against `rust/crates/rustproxy-http/src/connection_pool.rs`. Apply at
  the smartproxy repo root with `git apply`.

## One-line root cause
smartproxy's upstream connection pool proactively recycles pooled HTTP/2 & HTTP/3
connections at a hardcoded **120s age cap** (`MAX_H2_AGE`/`MAX_H3_AGE` in
`connection_pool.rs`) with no exemption for connections carrying active,
multiplexed WebSocket (Extended CONNECT) streams. Every 120s the aged upstream
connection is recycled and all WebSocket streams riding it drop at once —
synchronized, age-independent, and immune to keepalive/pings.

## Fix in one line
Track active multiplexed tunnels per pooled connection and exempt aged-but-active
connections from eviction (retire them from new checkouts; remove only once their
tunnels drain).
