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
5. **No per-row or per-field rights in v1.** If a future version adds them, they extend this law;
   they do not create a second, overlapping source of truth.
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

### Admin Out-of-Band Law

Setup and schema modification — create database, create entity, add user, browse the
catalog — are operations of **this service**, reachable over a connection and by a CLI
that needs no GUI. They are **never embedded in a consuming project's code**: an
application contains no schema setup, no migration script, and no file path. It
connects and uses what is already there.

## Current Proof Boundary

Nothing here is implemented; this lawbook establishes the accepted contract only. Owner
tests belong in `../../tests/darkbase-headless/` once source exists. No push, live
provider call, socket acquisition or credential use is implied by this document.
