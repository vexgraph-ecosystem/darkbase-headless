# darkbase-headless — Repo-Local Living Preferences

## Constitution Link

The complete [constitution](https://gist.github.com/vex-graph/4132a6c45cb6d3797c3e8eff2e94035a)
governs this repository. The binding law list is `gist.md`
(`../../../metadata/gists/preferences/gist.md`); `../../../PREFERENCES.md` is its full
rationale and is never ingested whole. Read the law first, then this file, the proof
lawbook `../../../tests/test-preferences.md` (slice one law with
`./tools/prefs test "<title>"`), the evidence ledger
`../../../metadata/gists/test-checklist.md`, and the actual implementation.

## Law Index (Binding Matrix)

| Law Title | Scope | Enforcement |
| :--- | :--- | :--- |
| Headless Service Boundary Law | R4 service scope and dependencies | Owner/build checks; no store internals, no UI |
| Client Never Opens The File Law | Ownership of the database file | Service is the only opener; clients hold connections |
| Session Token Law | Identity, tokens and credential handling | Token lifetime/revocation/redaction owner tests |
| Injected Transport Law | Socket ownership and bounded progress | fn-table seam, host-driven bounded polls, no threads |
| Request and Send Law | Wire shape, versioning and failure channel | Framing/status/version-mismatch owner tests |
| Admin Out-of-Band Law | Setup and schema modification | Setup absent from consumer code; admin ops owned here |
| Entity-Scoped Permission Law | Authorization model | Grants live on entities; role resolution and deny-by-default owner tests |
| Step-Up For Destructive Operations Law | Admin/destructive authorization | Password re-entry per destructive request; no cached credentials |
| Sensitive Field Law | Field-level protection | Sensitivity declared on the field; service-side enforcement; explicit withholding |
| Elevation Law (Time-Boxed Step-Up) | Presence proof and its window | Per-session elevation, bounded window, rate-limited, recovery codes |
| Authenticator Enrolment Law (TOTP) | Second factor and its primitives | RFC 6238 interop, gated enrolment, recoverable secret, replay rejection |

### Headless Service Boundary Law

R4 service for darkbase: owns **sessions, identity, the request/send surface and
the transport seam**; the only component that opens a database file. Borrows R2
contracts and the R3 store; never included by R3, never implements storage,
paging or the log — `darkbase-engine` (R3) owns those. No user interface; an R5
application consumes it. One-way dependencies: the store never learns what a
session is, the service never learns how a page is laid out.

### Client Never Opens The File Law

**The service is the only thing that opens the database.** A client holds a
**connection descriptor** — endpoint, credential, database id — never a path, even
on the same machine: the service's lifetime must stay independent of any client's. A
UI reading a `.vеxdb` directly is a defect: it makes the client's lifetime own the
store's and reduces this repository to decoration.

### Session Token Law

- **Authenticate once; verify a session per request:** the session token is checked
  per request, never a password — per-request hashing is a denial-of-service vector
  placing credentials on every message.
- **Two tokens, two lifetimes:** a short-lived **access token** in RAM dies with
  the process; a durable, long-lived (e.g. 90 days), revocable **refresh token**
  means a machine restart does not log everyone out.
- **The RAM token table is the hot set:** sessions and residency in one structure —
  pin on login, release on expiry or disconnect.
- **A token is an opaque value, never a pointer** (pointers are live-process
  addresses, not portable identifiers — the Stable Row and Variable Binding Law).
- **Never store a password:** store a salted, deliberately slow hash from the R2 CPU
  crypto contract (never invented) — and only the refresh token's hash.
- **Redaction.** Tokens and credential material never appear in logs, diagnostics or
  `toString` projections; comparisons of secret material do not early-exit.

### Entity-Scoped Permission Law

**Definition:** Authorization is **entity-scoped**: a **role is only a label** (no
rights), a **grant** — a `(roleId, entity, rights)` row with rights a bitmask (at
minimum READ, WRITE, DELETE) — is the only place a permission lives; a request
is authorized by resolving user → roles → grants for the target entity against the
right the operation needs. (Rationale: contradicting role and overwrite grants make
every audit a resolver simulation.)

**The Rule:**
1. Rights attach to entities, never to roles; adding a right to a role is a defect.
2. Deny by default: unknown user, role, entity or right rejects with an explicit
   status, never a silent allow.
3. One grant per `(role, entity)`; re-registering replaces, never accumulates.
4. Every catalog object (database, entity, user, role) carries a declared, stable
   id distinct from its name — renaming never breaks a grant or audit trail.
5. Field sensitivity is not a right: a flag on the field (the Sensitive Field Law)
   requiring elevation; grants stay on entities — never a second source of truth.
6. Authorization is per request, from the session's user, never cached across a
   grant change.

### Step-Up For Destructive Operations Law

**Definition:** Destructive or shape-changing operations require the **password
again** in that request; a session token is never sufficient alone. (Rationale: the
token proves *who you are*; the password proves *you are still the one holding
it*.)

**The Rule:**
1. Password in the request is required at minimum for: dropping an entity, deleting
   rows, changing grants, user creation/removal, token rotation/revocation.
2. Verify, then discard: checked against the stored slow hash, dropped immediately —
   never into the session, log, projection or diagnostic.
3. A failed step-up refuses with an explicit status and is auditable; repeated
   failures are rate-limited; a burst is a signal.
4. Reads and ordinary writes never require it — the cost is paid where the damage
   is.
5. The transport decision is load-bearing: a step-up password crosses the wire, so
   plaintext transport is *worse* than no step-up; step-up requires TLS or an
   explicit, documented trusted-network-only claim.
6. Admin overhead is by design — the cost of a breach being survivable.

### Sensitive Field Law

**Definition:** A field may be declared **sensitive**; reading or writing it
requires an **elevated** session (the Elevation Law). (Rationale: the schema is the
single answer to "is this protected?" — a client-side hide is a display choice, not
a control.)

**The Rule:**
1. Sensitivity is a flag on the `Field`, like `offset`, `size` and `KEY` — never a
   per-request option or client-side filter.
2. The service enforces it: non-elevated sensitive bytes never leave the process.
3. A direct read without elevation refuses with a *distinct* status, so a UI can
   ask for elevation rather than a generic denial.
4. A whole-row projection never silently drops it: it refuses, or includes the
   field **explicitly marked as withheld** — a reader must never mistake "withheld"
   for "empty" (silent redaction is the same defect as silent exhaustion).
5. Sensitivity gates writing too, by the same rule.
6. No per-row sensitivity in v1; a future version extends this law — the
   declaration stays on the field, not a second table.

### Elevation Law (Time-Boxed Step-Up)

**Definition:** **Elevation** is a per-session, time-boxed state proving a human is
present *now*: password plus authenticator code, a bounded window (**15 minutes**
default, configurable). (Rationale: the bounded window is the same trade `sudo`
makes; presence is what a stolen token cannot fake.)

**The Rule:**
1. Per-session, never global; elevating one session must not elevate another.
2. Password + authenticator, verified once, then discarded (Step-Up For Destructive
   Operations Law); the code is checked against a window, never stored.
3. Time-boxed, default 15 minutes, expiring silently to ordinary access; the code
   rotates on its own schedule — the window is the elevation's, not the code's.
4. Required for destructive operations (drop, delete, grants, users, tokens) and
   sensitive-field read or write.
5. Not required for ordinary reads and writes.
6. Failed attempts are rate-limited and auditable; a burst is a signal.
7. Recovery must exist: one-time backup codes issued at enrolment — "I lost my
   phone" must not mean "I lost my database".
8. The authenticator secret is recoverable by the service (unlike a password
   hash), so the store is part of the trust boundary — stated wherever described.

### Authenticator Enrolment Law (TOTP)

**Definition:** The second factor is **TOTP** (RFC 6238 over RFC 4226): a shared
per-user secret plus a time-derived 6-digit code. **No vendor integration**;
enrolment shows the setup URI once (as QR and as base32 text) and verification
recomputes the code from the stored secret. (Rationale: the interop surface is the
RFC and the setup URI, stable because it is owned by no one.)

**The Rule:**
1. Interop is the RFC, not a vendor: HMAC-SHA1, 6 digits, 30-second step, the
   `otpauth://totp/<issuer>:<account>?secret=<base32>&issuer=<issuer>` setup URI —
   no per-app code path, ever.
2. Enrolment is itself protected: enrolling or replacing an authenticator requires
   the password (Step-Up For Destructive Operations Law) — a stolen session that
   could silently enrol an attacker's device would permanently *own* step-up for
   the account.
3. The secret is stored so the service can read it (unlike a password hash): the
   store is part of the trust boundary — stored **encrypted**, key outside the
   database file, stated wherever described.
4. Recovery codes are issued at enrolment, shown exactly once, stored only as
   **hashes**, each **single-use**.
5. Verification accepts a bounded window (default ±1 step, for clock skew) and
   **rejects replay**: the last accepted timestep per user is remembered; a code
   at or before it is refused for its validity — a 30-second code with
   tolerance is otherwise reusable for up to ~90 seconds.
6. Failed verifications are rate-limited and auditable, like every elevation
   failure.
7. Primitives are borrowed, never invented: HMAC and the constant-time compare
   come from the R2 CPU crypto contract; base32 and the QR *encode* are pure
   format algorithms (Pure Format Algorithm Law); drawing the QR
   belongs to whichever surface shows it (terminal cells, or an image).
8. No connector is required: an SMS or e-mail fallback *would* be an R3 connector
   shape, weaker than TOTP — never the default path.

### Injected Transport Law

The transport is an **opaque handle plus a `{connect, poll, send, close}`
fn-table**, driven by the host's bounded pump — no owned sockets, no dedicated
threads, no unbounded waits (the Bounded Wait Law). In-process/embedded use only
skips the network; every client shares one request path. R3 may not include this
repository; R5 consumes it. A transport that blocks is a defect: a persistent push
connection must never become a polling loop.

### Request and Send Law

Requests and responses are **length-prefixed**, carry an explicit **protocol
version**, and answer with an **explicit status code** as the observable failure
channel; a version mismatch is refused loudly, never silently reinterpreted. **One
request at a time per connection** — multiple connections instead of multiplexing,
until ordering guarantees are designed.

First envelope: `protocol/headless_protocol.{h,c}`, normative layout in
`docs/wire-protocol.md`; V1: four-byte length prefix, then
version/kind/operation/status/request-id and opaque payload, explicitly
little-endian. Frames are bounded at 65536 bytes (hostile input safety), host
backing selected cold; a connection owner admits one frame and retains coalesced
tail bytes with the caller. Malformed/oversize/truncated streams fail closed until
an explicit new-stream reset; never scan for a new header. An admitted envelope is
not authentication or authority to execute; statuses are vocabulary with no
dispatcher or admin action connected yet. The host sends a full response before
release and owns deadlines, transport security and connection authorization; no
real socket or TLS proof is implied.

Frame backing, counts and ready/failure state mutate only through
constructor/feed/release/reset — a scoped managed accessor exception under the
Conflict Triage Law and Single Class Per File Law (Java Law), preserving
initialized-byte and borrow truth. Safe getters and redacted projections exist;
direct field mutation is outside contract; capacity refusals report once/count
always until reset.

### Admin Out-of-Band Law

Setup and schema modification — create database, create entity, add user, browse
the catalog — are operations of **this service**, reachable over a connection and
by a CLI needing no GUI, never **embedded in a consuming project's code**: an
application contains no schema setup, migration script or file path, and uses
what is already there.

## Current Proof Boundary

`src/service/service.{h,c}` is an initial offline service owner: one engine
Database; private admin row schema registered before loading; request admission
closed at startup; one-time bootstrap, password + TOTP login, TOTP replay-step
persistence, forced password replacement and explicit request-gate enable/disable.
The TOTP secret crosses a borrowed secret-store callback only; that callback must
be backed by secure OS storage, no plaintext fallback. Registered owner
`../../../tests/darkbase-headless/service/service_test.c` passed on macOS arm64
through the workspace graph; claims are scoped to that test, and save/load failure
injection and Keychain behavior are unproved.

It uses the engine's flat snapshot API for admin/replay/gate state; unimplemented:
WAL-backed mutation/audit direction, network transport, per-client
access/refresh sessions, CRUD, triggers and redacted audit — a process-local
authenticated boolean is not a client session or network authorization. Gaps:
Keychain integration, startup/save failure injection, diagnostic assertions,
sanitizers, non-macOS platforms; no push, live provider call, socket acquisition or
real credential use is implied.

The transport-independent protocol owner adds byte-level framing proof separately
(strict optimized and ASan/UBSan: hostile headers, splits, EOF, bounds/exhaustion,
preservation, recovery); transport and real network request execution remain
unimplemented.
