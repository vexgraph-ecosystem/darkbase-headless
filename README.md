# darkbase-headless — the R4 darkbase service

The headless service for [darkbase-engine](../../drivers/darkbase-engine): it owns
database startup and file access, one admin's bootstrap/authentication and the
current process-local request gate. The wire envelope exists; sessions, transport
driving and request execution remain planned. The graphical/TUI application is a **client of this**, not a second owner
of the store.

## Current State

**Role:** R4 interface. Current service source borrows R2 (`vexspoke`,
`relational-engine`) and R3 `darkbase-engine`; API Haven and Sesh remain planned
dependencies. R5 applications are intended consumers.

**A first offline service slice is implemented and its registered owner test has
passed on macOS arm64.** `src/service/service.{h,c}` owns one engine `Database`, registers a
private admin row before startup load, bootstraps the one-time `admin` /
`Password123!` credential, and requires password + TOTP for login. Startup keeps
request admission closed; the bootstrap password must be replaced before an
admin may explicitly enable admission. TOTP replay state and the enable preference
are written through the engine's current snapshot API before the corresponding
operation succeeds. A borrowed secure-secret callback seam has no plaintext
fallback; platform Keychain integration is not implemented here.

The service currently exposes only an in-process admission boolean, not a network
listener, per-client sessions/tokens, CRUD dispatcher, triggers,
or durable audit. TOTP/password/admin policy data currently shares the legacy
snapshot; the specified WAL-backed mutation/audit path is not integrated. The
owner test exercises bootstrap, login, replay, password-change gating and restart
using an in-memory fake secret store. Save/load failure injection, diagnostic
assertions, sanitizers and live Keychain proof remain outstanding.

**Wire envelope v1 is implemented separately** in
`src/protocol/headless_protocol.{h,c}`: little-endian length/version/kind/operation/
status/request-ID fields, opaque payloads, partial receive handling, retained
coalesced tails and one admitted frame per owner. Malformed, oversize, wrong-version
and truncated streams fail closed; encoding preserves output on rejection. The
registered protocol owner and the standalone optimized/ASan/UBSan runner pass on
macOS arm64. See [wire-protocol.md](docs/wire-protocol.md) for the normative layout
and borrowing contract. This codec owns no sockets, threads, waits or allocation,
and invokes no service/admin action. Valid framing is not authorization.

Still planned: connection descriptors; access/refresh session tokens and per-request
authorization; injected transport driving and service dispatch; admin catalog operations;
CRUD, triggers and redacted durable audit; entity-scoped permissions, sensitive-field
gating, elevation and step-up, TOTP enrolment/recovery codes.

## Scope and Limitations

- **Covers today:** service-owned database startup/load, one admin bootstrap/login,
  TOTP replay watermark, forced password replacement and explicit process-local
  request admission, plus a separate versioned request/response envelope codec.
  The broader service surface is future work.
- **Does not cover:** the store itself (pages, log, catalog — that is `darkbase-engine`,
  R3); any UI (that is the R5 client); network transport, client sessions, CRUD,
  triggers or WAL-backed audit. The wire codec remains offline; no listener, TLS,
  host pump, partial-send adapter or network execution has been integrated.
- **The one rule to keep it honest:** a **client never opens the file** — not even on
  the same machine. The moment a UI reads a `.vexdb` directly, the UI's lifetime owns
  the store's and this repository's reason to exist disappears.
- **Platform:** the registered offline owner passed on macOS arm64. Windows and Linux
  runtime behavior is untested; live platform secret-store behavior is unproved.
- **Protocol proof:** strict optimized and ASan/UBSan codec owners cover split/EOF/
  hostile headers, capacity refusal/reset and bounded projections, not real TCP/TLS,
  credential schemas, deadlines, connection authorization or leak instrumentation.
- **Lawbook:** [`darkbase-headless-preferences.md`](darkbase-headless-preferences.md).
