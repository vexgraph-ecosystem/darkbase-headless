# SECURITY.md — darkbase-headless

> **Internal invariant set, not a public disclosure channel.** See the
> [vexgraph security baseline](../../../SECURITY.md) for everything inherited
> here. This file states only this repository's own surface.

**Role:** the R4 service for darkbase. It owns sessions, identity, the
request/send surface and the transport seam — and **it is the only component
that opens a database file**. It borrows R2 contracts and the R3 store; the store
never learns what a session is.

## Secrets

- **A password is never stored.** Only a salted, deliberately slow hash, borrowed
  from the R2 crypto contract — never an invented one.
- A refresh token is stored **only as a hash**. An access token lives in RAM,
  dies with the process, and is an opaque value, never a pointer.
- The **TOTP secret is recoverable by the service** (unlike a password hash), so
  the store is part of the trust boundary: it is stored encrypted, with the key
  held outside the database file, and that is stated wherever this is described.
- Tokens and credential material never appear in a log, diagnostic or `toString`
  projection, and comparison of secret material does not early-exit.

## Authorization

- **Entity-scoped grants.** A role is a label with no rights of its own; a grant
  is `(role, entity, rights)` and is the only place a permission lives. Deny by
  default, one grant per (role, entity), and authorization recomputed per request
  — never cached across a grant change.
- **Step-up for destruction.** Dropping, deleting, grant/user/token changes and
  sensitive-field access require the password *again*, verified and then
  discarded, never cached in the session.
- **Elevation is time-boxed** (default 15 minutes, per-session) and needs
  password plus authenticator. Enrolment is itself gated by the password —
  otherwise a stolen session could permanently own step-up.
- **TOTP replay is rejected** within a bounded window, and the last accepted
  timestep is remembered per user.

## Failed attempts are throttled and counted

Repeated failures are rate-limited and auditable, and a burst is a signal rather
than noise. A security-motivated delay at such a site is marked
`;;FREEZE("bound, identity")` and obeys all six rules in
`annotation/security.h`: cold path only, bounded by a named constant whose
ceiling a test asserts, never holding a lock, never inside a loop whose count
comes from the request, always paired with a counter, and never a substitute for
the access check itself.

## Transport

**A step-up password crosses the wire**, so a plaintext transport is *worse* than
no step-up at all — it would expose the very credential that protects
everything. Step-up therefore requires TLS, or an explicit documented
trusted-network-only claim. The transport is injected by the host, which owns
deadlines, transport security and connection authorization; this service is not a
socket owner.

## Limits

- The bootstrap credential and its removal are unfinished work; enrolment,
  recovery, durable replay state and per-client sessions are not all proved.
- **No WAL-backed mutation/audit path yet** — the current slice uses the engine's
  flat snapshot API, so audit durability is not claimed.
- A process-local authenticated boolean is **not** a client session and not a
  network authorization system.
- Keychain integration, save/load failure injection, sanitizers and non-macOS
  platforms remain unproved.
