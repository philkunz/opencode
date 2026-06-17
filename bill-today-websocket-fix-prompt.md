# Fix prompt: WebSocket connections through smartproxy drop every 120s (pooled HTTP/2 & HTTP/3 connection age cap)

## Repository to fix
`@push.rocks/smartproxy` (source: `code.foss.global/push.rocks/smartproxy`), Rust crate
`rust/crates/rustproxy-http`. Confirmed against v27.17.3.

## Symptom
Any long-lived WebSocket proxied through smartproxy (e.g. `wss://bill.today/`, fronted by
serve.zone `dcrouter` → `coretraffic`) is force-closed (`code 1006`, no close frame) on a
**120-second, wall-clock-aligned cadence**. The browser/client auto-reconnects, so users see a
reconnect "every ~2 minutes." Application- or server-level keepalives do **not** help (the
backend already sends RFC6455 pings every 30s).

## Root cause (confirmed in source)
`rust/crates/rustproxy-http/src/connection_pool.rs`:

```rust
const EVICTION_INTERVAL: Duration = Duration::from_secs(30);
const MAX_H2_AGE: Duration = Duration::from_secs(120);   // line 28
const MAX_H3_AGE: Duration = Duration::from_secs(120);   // line 30
```

The pool reuses upstream keep-alive connections. HTTP/2 and HTTP/3 (QUIC) connections are
**multiplexed** — many client requests/streams share one upstream connection. WebSocket upgrades
are carried as multiplexed streams over these connections:
- H2/H3 fronts via RFC 8441 / RFC 9220 Extended CONNECT,
- and upstream WebSocket backends use the **H3** Extended CONNECT path
  (`WsBackendEndpoint::H3 { send, recv }` in `proxy_service.rs`), riding a pooled QUIC connection.

The background eviction loop (`connection_pool.rs::eviction_loop`, lines ~373–397) and the
checkout paths (`checkout_h2` line 177, `checkout_h3` line 248) recycle any pooled connection once
`created_at.elapsed() >= MAX_H2_AGE/MAX_H3_AGE` (120s) — **with no exemption for connections that
carry active, long-lived multiplexed streams (WebSocket tunnels).** When the aged upstream
connection is recycled, every WebSocket stream multiplexed on it is dropped **simultaneously**,
regardless of each stream's own age.

This is why the drop is:
- **synchronized across independent client connections** (they share one pooled upstream conn),
- **age-independent per stream** (a 16s-old WS dies with a 56s-old WS),
- **phase-locked to wall-clock at 120s** (it tracks the pooled connection's `created_at`),
- **immune to keepalive/pings** (eviction is by connection age, not idle time).

### Why other theories are wrong
- It is **not** `inactivityTimeout`/`socketTimeout` (dcrouter sets these to 120_000). Those are
  per-connection and reset on activity; the 30s backend pings would prevent them, and they cannot
  kill a 16s-old stream at the same instant as a 56s-old one. The dcrouter `inactivityTimeout:
  120_000` value coincidentally equals `MAX_*_AGE`, which made it look like an inactivity timeout.
- `MAX_H2_AGE`/`MAX_H3_AGE` are **hardcoded consts**, so no `dcrouter`/`coretraffic`
  `ISmartProxyOptions` change (e.g. `keepAliveTreatment: 'immortal'`, larger `inactivityTimeout`)
  can fix it. The fix must be in smartproxy.

## Empirical evidence (live, against `wss://bill.today/`)
- Staggered clients opened at +0s/+20s/+40s all closed at the **same wall-clock instant**
  (10:18:32.45, sub-ms apart), at ages 56.2s / 35.7s / 16.1s — across two separate processes.
- A reconnecting client: cycle #1 (opened mid-interval) lived 56.3s; cycle #2 (opened just after a
  tick) lived **119.9s**; ticks were exactly 120.0s apart (10:18:32 → 10:20:32).
- Backend sends WS pings at 30s/60s — connection still dropped. Confirms not idle-based.

## Should any prior commit be reverted? — No
The 120s age cap is deliberate and correct **for stateless HTTP connection pooling**; the bug is
that it is applied indiscriminately to connections carrying active WebSocket tunnels. History of
`connection_pool.rs`:

| date | commit | MAX_H2_AGE | rationale |
|---|---|---|---|
| 03-11 | `5271447` | 120 | introduce age cap; "evict stale pooled senders … avoid 502s" |
| 03-15 | `a9dbccf` | 300 | "tune HTTP/2 connection lifetimes" + keep idle tracking alive while streaming |
| 03-16 | `d5e08c8` | 120 | "120s is well within typical server GOAWAY windows (nginx ~60s idle, envoy ~60s)" |
| 03-19 | `4fb91cd` | 120 (+H3 120) | add `MAX_H3_AGE` to match |

These all make sense for their stated purpose: the cap proactively recycles pooled connections
**before** a backend's idle GOAWAY would, avoiding reuse-race 502s. **Do not revert any of them:**
- Reverting `d5e08c8` (→300s) would only move the WebSocket reconnect to every ~5 min and would
  re-introduce the GOAWAY/502 staleness risk it was added to prevent.
- The WebSocket-aware commits (`a9dbccf` "keep idle tracking alive during streaming", `2cb284f`
  "keep idle h3 websocket tunnels alive", 27.17.3's QUIC keep-alive) only ever addressed the
  **idle** path (IDLE_TIMEOUT / 45s QUIC max-idle). None addressed the **age cap** conflicting with
  active upgraded tunnels — which is the actual gap.

The fix is therefore **additive** (exempt active-upgrade-bearing connections from the age cap), not
a revert and not a value change. Note also the design tension to preserve: a pooled connection
carrying a 24h-`ws_max_lifetime` WebSocket tunnel is currently recycled out from under it at 120s.

## Required fix
Do **not** recycle/evict a pooled H2 or H3 connection that still has active upgraded
(WebSocket/Extended CONNECT) streams or other long-lived in-flight streams. Concretely:

1. **Track active multiplexed streams per pooled connection.** The proxy already has an
   `active_upgrades: Arc<AtomicU64>` counter (`proxy_service.rs:54`, incremented around
   `spawn_ws_tunnel`). Associate a live-stream / active-upgrade count with each `PooledH2` /
   `PooledH3` entry (e.g. store an `Arc<AtomicU64>` in the pool entry, incremented when a WS/long
   stream opens on that connection and decremented on close).

2. **Exempt connections with active streams from the age cap.** In `eviction_loop`,
   `checkout_h2`, and `checkout_h3`, when `created_at.elapsed() >= MAX_*_AGE`:
   - if the connection has **zero** active multiplexed/upgraded streams → recycle as today
     (stop handing it out for new requests and let it close);
   - if it has active streams → **do not force-close**; instead stop using it for *new* checkouts
     (so new requests get a fresh connection) but keep the existing connection alive until its
     active streams finish on their own (governed by the WS-specific `ws_inactivity_timeout` /
     `ws_max_lifetime`, which already default to 1h/24h in `proxy_service.rs`).

3. The age cap's purpose (avoid unbounded reuse of stale pooled connections / force periodic
   re-resolution) is preserved for idle/stateless traffic — only long-lived multiplexed tunnels
   are spared a mid-stream kill.

### Exact close mechanism (confirmed)
The WebSocket-over-H3 backend path is `ws_connect_backend_h3` (`proxy_service.rs:4661`). It opens
the RFC 9220 Extended CONNECT stream from a **clone** of the pooled `send_request`, then drops its
local copy — so the pool's stored `send_request` is the **last application `SendRequest` handle**.
When `checkout_h3`/`eviction_loop` removes the entry at `MAX_H3_AGE`, that last `SendRequest` is
dropped, the h3 client closes the QUIC connection, and the live WS stream dies (client sees 1006).
The fix keeps the pool entry (and thus its `send_request`) alive while `active.count() > 0`, so the
connection is not closed out from under an active tunnel.

### Concrete patch (this directory)
`smartproxy-connection-pool-ws-age-exemption.patch` — apply with
`git apply smartproxy-connection-pool-ws-age-exemption.patch` at the smartproxy repo root. It
changes only `rust/crates/rustproxy-http/src/connection_pool.rs`:
- adds `ActiveStreams` / `ActiveStreamGuard` (an `Arc<AtomicU64>` refcount + RAII guard);
- adds an `active: ActiveStreams` field to `PooledH2` / `PooledH3`;
- `checkout_h2`/`checkout_h3`: an aged-out connection is no longer handed to new requests, but is
  only physically removed when `active.count() == 0` (dead connections still evicted immediately);
- `register_h2`/`register_h3`: now return `(generation, ActiveStreams)`;
- `eviction_loop`: aged H2/H3 entries with active tunnels are retained until they drain.

This is the **core mechanism fix**. It compiles in isolation but changes the `checkout_*`/
`register_*` signatures, so the call sites below must be wired (mechanical):

#### Required call-site wiring (`proxy_service.rs`)
- `checkout_h3` callers: `:1905` (regular forward — bind/ignore the new 4th element) and **`:4681`
  inside `ws_connect_backend_h3` (the WS path)**.
- `checkout_h2` caller: `:2086` (bind/ignore the new 3rd element).
- `register_h2` callers: `:3145`, `:3381`, `:3580` — now returns a tuple `(g, _active)`.
- `register_h3` callers: `:4829`, `:5724` — `:4829` is in `ws_connect_backend_h3` (the WS path).
- **In `ws_connect_backend_h3`:** capture the `ActiveStreams` from both the pool-hit (`checkout_h3`)
  and fresh (`register_h3`) branches; call `let guard = active.guard();` and thread `guard` out
  through `finish_websocket_upgrade` (`:4844`) into `spawn_ws_tunnel` (`:4933`/`:4994`). Store it
  next to the existing `_ws_upgrade_guard` / `_ws_route_connection_guard` RAII guards inside the
  tunnel task (`spawn_ws_tunnel`, ~`:779–782`) so it is held for the tunnel's whole lifetime and
  dropped when the tunnel ends. The pattern already exists there — mirror it.
- (H2 backend WebSockets do not exist in this codebase — `WsBackendEndpoint` is only `Io`/`H3` — so
  the H2 `active` plumbing is inert today but kept for symmetry / future long-lived H2 streams.)

### Acceptance criteria / test plan
- Add/extend an integration test: open a WebSocket through the proxy, keep it **idle past 240s**
  (two age-cap intervals), with only periodic pings — assert it stays open (no 1006) and that
  regular pooled HTTP requests still recycle their underlying connection at ~120s.
- Open N WebSockets multiplexed on one upstream H3 connection; assert none is dropped at the 120s
  boundary while streams are active.
- Confirm a fresh upstream connection is used for *new* requests after the old one passes 120s
  (no indefinite reuse of an aged connection for new work).

## Secondary mitigations (optional, do not replace the smartproxy fix)
- `dcrouter`/`coretraffic`: nothing config-level can fix this (age caps are hardcoded). Track the
  smartproxy fix and bump the dependency once released.
- Client resilience (already present): TypedSocket auto-reconnects; the visible "reconnect every
  2 min" is the symptom, not a fix.

## Note for the implementer to verify in Rust
The empirical 120s teardown is conclusive, and the `MAX_*_AGE = 120s` constants are the only
matching mechanism. Confirm the precise close propagation (eviction `remove` dropping the last
pool ref vs. a fresh connection replacing the aged one in `enforce_*_key_limit` / driver teardown)
and ensure the exemption covers whichever path actually severs active streams.
