# SECURITY.md — darkbase-headless

> **Internal invariant set, not a public disclosure channel.** See the
> [vexgraph security baseline](../../../SECURITY.md) for everything inherited
> here. This file states only this repository's own surface.

**Role:** the R4 service for darkbase. It owns sessions, identity, the
request/send surface and the transport seam — and **it is the only component
that opens a database file**. It borrows R2 contracts and the R3 store; the store
never learns what a session is.

## Secrets

- **A password is never stored** — verified: PBKDF2-SHA256 over a random salt
  with the derivation cost recorded beside the hash (`passwordSalt`,
  `passwordHash`, `iterations`). There is no plaintext path.
- **The TOTP secret never lands in the database file** — it crosses an injected
  secret-store seam. The service *requires* that seam to be backed by secure OS
  storage with the key outside the database; the service does not encrypt it
  itself, so that guarantee is only as strong as the host's store, and Keychain
  integration is unproved.
- **Per-client tokens are NOT implemented.** `refresh` appears nowhere in `src/`:
  a process-local authenticated boolean is not a client session and not network
  authorization. The Session Token Law describes the target, not the current
  state.
- Credential material never appears in a log, diagnostic or `toString`
  projection, and comparison of secret material does not early-exit.

## Authorization — the model, and what actually exists

**Implemented today:** one-time bootstrap, password + TOTP login, **TOTP replay
rejection** (a durable `lastTotpStep` watermark), forced password replacement, and
an explicit request-gate enable/disable.

**Required, not yet implemented:** entity-scoped grants, step-up for destructive
operations, time-boxed elevation, sensitive-field enforcement and per-client
sessions. The model those must satisfy, when they land:

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

Until those exist, this service authenticates but does not yet authorize.

## Failed attempts are NOT yet throttled

**Verified against the implementation:** repeated failures are **not**
rate-limited, not counted and not delayed. `HeadlessService_adminLogin` rejects
with a diagnostic and nothing else — there is no counter, no lockout and no
backoff anywhere in `src/`. A burst of guesses is currently indistinguishable
from a burst of typos.

That is a **required gap, not a description**. When it lands it is the first
honest `;;FREEZE` site in the ecosystem: a security-motivated delay marked
`;;FREEZE("bound, identity")`, obeying all six rules in `annotation/security.h`
— cold path only, bounded by a named constant whose ceiling a test asserts,
never holding a lock, never inside a loop whose count comes from the request,
always paired with a counted refusal, and never a substitute for the access
check. Until then, no rate limiting may be claimed for this service, here or
anywhere it is described.

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
