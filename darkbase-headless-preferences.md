# darkbase-headless — Repo-Local Living Preferences

## Constitution Link

The complete [constitution](https://gist.github.com/vex-graph/4132a6c45cb6d3797c3e8eff2e94035a)
governs this repository. Workspace source: `../../gist/PREFERENCES.md`.
Read it first, then this file, `../../tests/test-preferences.md`, the current
`../../gist/test-checklist.md`, and the actual implementation.

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

This repository is the R4 service for darkbase: it owns **sessions, identity, the
request/send surface and the transport seam**, and it is the only component that opens
a database file. It borrows R2 contracts and the R3 store; it is never included by R3,
and it never implements storage, paging or the log — those belong to `darkbase-engine`
(R3). It contains no user interface; an R5 application consumes it. The dependency
direction is one way: the store never learns what a session is, and the service never
learns how a page is laid out.

### Client Never Opens The File Law

**The service is the only thing that opens the database.** A client holds a
**connection descriptor** — endpoint, credential, database id — and never a path. This
holds even when client and service are the same machine, because the entire point of
the separation is that the service's lifetime is independent of any client's. A UI that
reads a `.vеxdb` directly is a defect, not a convenience: it makes the client's lifetime
own the store's and reduces this repository to decoration.

### Session Token Law

- **Authenticate once; verify a session per request.** Per-request verification checks
  a session token, never a password. Hashing a password per request is a denial-of-
  service vector and puts credentials on every message.
- **Two tokens, two lifetimes.** An **access token** lives in RAM, is short-lived, and
  dies with the process — which is correct: "who is online right now" is exactly the
  resident set. A **refresh token** is durable, long-lived (e.g. 90 days), and
  revocable, so a machine restart does not log everyone out.
- **The RAM token table is the hot set.** Sessions and residency are one structure:
  pin on login, release on expiry or disconnect.
- **A token is an opaque value, never a pointer.** Pointers are live-process addresses,
  not portable identifiers (the Stable Row and Variable Binding Law).
- **Never store a password.** Store a salted, deliberately slow hash borrowed from the
  R2 CPU crypto contract; never invent a hash. Store only the hash of a refresh token.
- **Redaction.** Tokens and credential material never appear in logs, diagnostics or
  `toString` projections. Comparisons of secret material do not early-exit.

### Entity-Scoped Permission Law

#### Definition:
Authorization is **entity-scoped**. A **role is only a label** attached to a user — it carries
no rights of its own. A **grant** is a row of `(roleId, entity, rights)`, where rights is a
bitmask (at minimum READ, WRITE, DELETE), and it is the only place a permission lives. A
request is authorized by resolving user → roles → that user's grants for the target entity,
and comparing against the right the operation needs.

#### The Why:
One permission, one place. The model this deliberately is *not* is Discord's, where a role and
a per-object overwrite are two things that can contradict each other and the answer depends on
evaluation order. When two places can grant the same right, every audit becomes a simulation of
the resolver. Here there is exactly one place, which makes "who can do what to this table" a
single read, not a reasoning exercise.

#### The Rule:
1. **Rights attach to entities, never to roles.** A role is a label; adding a right to a role
   is a defect.
2. **Deny by default.** No grant means no access. An unknown user, role, entity or right rejects
   with an explicit status — never a silent allow.
3. **One grant per (role, entity).** Re-registering replaces; it never accumulates.
4. **Every catalog object carries a declared, stable id** — database, entity, user, role — distinct
   from its name, so renaming never breaks a grant or an audit trail. Ids are declared, never
   derived from position.
5. **Field sensitivity is not a right.** It is a flag on the field (the Sensitive Field Law) that
   requires elevation; grants stay on entities. Two different concerns, one place each — never a
   second, overlapping source of truth for the same question.
6. **Authorization is per request**, computed from the session's user, and never cached across a
   grant change.

### Step-Up For Destructive Operations Law

#### Definition:
Operations that destroy or reshape data require the **password again**, carried in that request.
A valid session token is never sufficient on its own. The password is verified against the stored
slow hash and then **discarded** — never cached, never logged, never stored in the session.

#### The Why:
A session token is the thing most likely to be stolen: it sits in RAM, it travels on every
request, and it lasts. If possession of that token is enough to drop a table, then any breach of
the token table — or any stolen session — hands over the ability to destroy the data itself. The
token proves *who you are*; the password proves *you are still the one holding it*. Keeping the
second proof off the wire until the moment it is needed is what makes a breach survivable.

#### The Rule:
1. **Destructive and shape-changing operations require the password in the request.** At minimum:
   dropping an entity, deleting rows, changing grants, creating or removing users, and rotating
   or revoking tokens.
2. **Verify, then discard.** The password is checked against the stored slow hash and dropped
   immediately. It is never written into the session, the log, a projection or a diagnostic.
3. **A failed step-up refuses with an explicit status** and is itself auditable. Repeated failures
   are rate-limited; a burst of them is a signal, not noise.
4. **Reads and ordinary writes never require it.** Step-up is for destruction, not for use — the
   cost is paid exactly where the damage would be.
5. **The transport decision is load-bearing.** A step-up password crosses the wire, so a plaintext
   transport is *worse* than no step-up at all: it would expose the credential that protects
   everything. Step-up requires either TLS or an explicit, documented trusted-network-only claim.
6. **Admin overhead is by design, not an oversight.** Extra friction on destructive operations is
   the intended cost of a breach being survivable.

### Sensitive Field Law

#### Definition:
A field may be declared **sensitive** — a flag on the field itself, exactly like `KEY`, declared
beside offset and size. Reading or writing a sensitive field requires an **elevated** session
(the Elevation Law). The **service** enforces it at the boundary: sensitive bytes are never sent
to a session that is not elevated.

#### The Why:
Protection belongs where the data is, so it cannot drift from it. Declaring sensitivity on the
field means the schema is the single answer to "is this protected?", the same way the layout is
the single answer to "where is this?" — no separate permissions table to fall out of sync, no
per-request option to forget. And enforcement has to be service-side: a client that hides a field
locally is a display choice, not a control, and anyone talking to the service directly sees it
anyway.

#### The Rule:
1. **Sensitivity is a flag on the `Field`**, declared like `offset`, `size` and `KEY`. Never a
   per-request option, never a client-side filter.
2. **The service enforces it.** Bytes of a non-elevated sensitive field never leave the process.
3. **A direct read of a sensitive field without elevation refuses** with a *distinct* status, so a
   UI can ask for elevation rather than showing a generic denial.
4. **A whole-row projection never silently drops it.** It either refuses, or includes the field
   **explicitly marked as withheld**. A reader must never mistake "withheld" for "empty" — silent
   redaction is the same defect as silent exhaustion.
5. **Sensitivity gates writing too.** Setting a sensitive field requires elevation by the same rule.
6. **No per-row sensitivity in v1.** If a future version adds it, it extends this law; the
   declaration stays on the field, not in a second table.

### Elevation Law (Time-Boxed Step-Up)

#### Definition:
**Elevation** is a per-session, time-boxed state proving a human is present *now*. It is obtained
with the password plus an authenticator code, lasts a bounded window (**15 minutes** by default,
configurable), and unlocks destructive operations and sensitive-field access. When it expires, the
session returns to ordinary access with no further ceremony.

#### The Why:
Re-prompting on every single call makes a control unusable, and an unusable control protects
nothing — people route around it, or switch it off. A bounded window is the same trade `sudo`
makes (its default re-prompt interval is 15 minutes, for the same reason). What elevation actually
proves is *presence*: the session token says who you are, the elevation says a human is holding
the device right now. That is the property a stolen token cannot fake.

#### The Rule:
1. **Per-session, never global.** Elevating one session must not elevate another.
2. **Password + authenticator**, verified once, then discarded (Step-Up For Destructive Operations
   Law). The code is checked against a window, never stored.
3. **Time-boxed, default 15 minutes**, and it expires back to ordinary access silently. The
   authenticator code itself rotates on its own shorter schedule; the window is the elevation's,
   not the code's.
4. **Required for:** destructive operations (drop, delete, grants, users, tokens) and sensitive-field
   read or write.
5. **Not required for:** ordinary reads and writes, so the friction lands only where damage or
   exposure would.
6. **Failed attempts are rate-limited and auditable**; a burst is a signal.
7. **Recovery must exist.** One-time backup codes are issued at enrolment, because "I lost my
   phone" must not mean "I lost my database".
8. **The authenticator secret is recoverable by the service** (unlike a password hash), so the
   store itself is part of the trust boundary. That must be stated wherever this is described,
   never implied away.

### Authenticator Enrolment Law (TOTP)

#### Definition:
The second factor is **TOTP** (RFC 6238 over RFC 4226): a shared per-user secret plus a
time-derived 6-digit code. There is **no vendor integration** — Google Authenticator, Authy,
Apple Passwords, 1Password and every other app implement the same two RFCs and the same
`otpauth://` setup URI. Enrolment shows that URI once (as QR and as base32 text); verification
recomputes the code from the stored secret.

#### The Why:
The reason one enrolment works in every app is that none of them is a service to connect to.
Treating "integrate with Authy/Google/Apple" as a connector problem would build phantom work:
there is no vendor API in the middle, no OAuth, and nothing to poll. The interop surface is the
RFC and the setup URI, and that surface is stable precisely because it is not owned by anyone.

#### The Rule:
1. **Interop is the RFC, not a vendor.** HMAC-SHA1, 6 digits, 30-second step, the
   `otpauth://totp/<issuer>:<account>?secret=<base32>&issuer=<issuer>` setup URI. No per-app code
   path, ever.
2. **Enrolment is itself protected.** Enrolling or replacing an authenticator requires the
   password (Step-Up For Destructive Operations Law). A stolen session that could silently enrol
   an attacker's device would permanently *own* step-up for that account — this is the single
   most dangerous hole in the whole scheme, and it is closed by gating enrolment.
3. **The secret is stored so the service can read it** (unlike a password hash), which makes the
   store part of the trust boundary. It is therefore stored **encrypted**, with the key held
   outside the database file, and that fact is stated wherever this is described.
4. **Recovery codes are issued at enrolment**, shown exactly once, stored only as **hashes**, and
   each is **single-use**. "I lost my phone" must never mean "I lost my database".
5. **Verification accepts a bounded window** (default ±1 step, for clock skew) and **rejects
   replay**: the last accepted timestep per user is remembered, and a code at or before it is
   refused for the rest of its validity. A 30-second code with tolerance is otherwise reusable
   for up to ~90 seconds by anyone who saw it.
6. **Failed verifications are rate-limited and auditable**, like every other elevation failure.
7. **Primitives are borrowed, never invented.** HMAC and the constant-time compare come from the
   R2 CPU crypto contract; base32 and the QR *encode* are pure byte-to-byte format algorithms
   (Pure Format Algorithm Law) — a decoder already exists in spirit, an encoder is its sibling.
   Drawing the QR belongs to whichever surface shows it (terminal cells, or an image).
8. **No connector is required.** An SMS or e-mail delivery fallback *would* be a third-party API
   and therefore an R3 connector shape — but that is a weaker factor than TOTP and must not be the
   default path.

### Injected Transport Law

The transport is an **opaque handle plus a `{connect, poll, send, close}` fn-table**,
driven by the host's bounded pump — no owned sockets, no dedicated threads, no
unbounded waits (the Bounded Wait Law). In-process/embedded use is simply a transport
that skips the network, so every client shares one request path. R3 may not include
this repository; R5 consumes it. A transport that blocks is a defect: a persistent
push connection must never become a polling loop.

### Request and Send Law

Requests and responses are **length-prefixed**, carry an explicit **protocol version**,
and answer with an **explicit status code** as the observable failure channel. A
version mismatch is refused loudly, never silently reinterpreted — once client and
service version independently, a mismatched pair is a new failure mode that a
single-repository design never had. **One request at a time per connection**; multiple
connections instead of multiplexing, until ordering guarantees are designed.

The first envelope implementation is `protocol/headless_protocol.{h,c}`; the
normative byte layout is in `docs/wire-protocol.md`. V1 has a four-byte length
prefix followed by version/kind/operation/status/request-id and opaque payload,
all explicitly little-endian. Total frames are bounded at 65536 bytes for hostile
input safety; host backing is selected cold. A connection owner admits one frame
and retains coalesced tail bytes with the caller. Malformed/oversize/truncated
streams fail closed until an explicit new-stream reset; never scan for a new
header. An admitted envelope is not authentication or authority to execute.
Response statuses are vocabulary; no dispatcher or admin action is connected yet.
The host must send a full response before release and owns deadlines, transport
security and connection authorization. No real socket or TLS proof is implied.

Frame backing, counts and ready/failure state mutate through constructor/feed/
release/reset only, a scoped managed accessor exception under the Conflict Triage
Law and Single Class Per File Law (Java Law), preserving initialized-byte and
borrow truth. Safe getters and redacted projections exist; direct field mutation
is outside contract. Capacity refusals report once/count always until reset.

### Admin Out-of-Band Law

Setup and schema modification — create database, create entity, add user, browse the
catalog — are operations of **this service**, reachable over a connection and by a CLI
that needs no GUI. They are **never embedded in a consuming project's code**: an
application contains no schema setup, no migration script, and no file path. It
connects and uses what is already there.

## Current Proof Boundary

`src/service/service.{h,c}` implements an initial offline service owner: it owns one
engine Database, registers the private admin row schema before loading, keeps request
admission closed at startup, and supports one-time bootstrap, password + TOTP login,
TOTP replay-step persistence, forced password replacement and explicit request-gate
enable/disable. The TOTP secret crosses a borrowed secret-store callback only; that
callback must be backed by secure OS storage and there is no plaintext fallback.
The registered owner `../../tests/darkbase-headless/service/service_test.c` passed
on macOS arm64 through the workspace graph, covering bootstrap, login, replay,
password-change gating and restart with an in-memory fake secret store. These claims
remain scoped to that test; save/load failure injection and Keychain behavior are
unproved.

This slice uses the engine's existing flat snapshot API for admin/replay/gate state.
It does not implement the approved WAL-backed mutation/audit direction, network
transport, per-client access/refresh sessions, CRUD, triggers or
redacted audit. A process-local authenticated boolean is not a client session or a
network authorization system. Keychain integration, startup/save failure injection,
diagnostic assertions, sanitizers and non-macOS platforms remain proof gaps. No push,
live provider call, socket acquisition or real credential use is implied.

The transport-independent protocol owner adds byte-level framing proof separately
from the service. Its strict optimized and ASan/UBSan runner exercises hostile
headers, splits, EOF, bounds/exhaustion, preservation and recovery. Service-level
sanitizer/failure-injection gaps above are unchanged. Transport and real network
request execution remain unimplemented.
