# darkbase-headless — the R4 darkbase service

The headless client and service for [darkbase-engine](../../drivers/darkbase-engine):
it owns identity, sessions and the request/send surface, and it is the only thing that
opens a database file. The graphical/TUI application is a **client of this**, not a
second owner of the store.

## Current State

**Role:** R4 interface. Borrows R2 (`vexspoke`, `relational-engine`) and R3
(`darkbase-engine`, `api-haven`, `sesh`); is consumed by R5 applications.

**Nothing is implemented yet.** The repository holds its constitution link, lawbook,
LICENSE and ignore list — no source, no build target, no tests. Everything below is
specified intent, not delivered behavior.

Specified, not built: the connection descriptor (endpoint + credential + database),
the session model (access token in RAM, durable refresh token), per-request
verification, the request/send framing with an injected transport seam, and the admin
operations (create database, create entity, add user, browse catalog).

## Scope and Limitations

- **Covers:** the service surface — who is connected, what they may do, how a request
  is framed and answered, and the transport seam. It is the surface *every* darkbase
  client talks to.
- **Does not cover:** the store itself (pages, log, catalog — that is `darkbase-engine`,
  R3); any UI (that is the R5 client); and no wire protocol is implemented yet.
- **The one rule to keep it honest:** a **client never opens the file** — not even on
  the same machine. The moment a UI reads a `.vexdb` directly, the UI's lifetime owns
  the store's and this repository's reason to exist disappears.
- **Platform:** unproven. macOS arm64 is the only lab; Windows and Linux are untested.
- **Lawbook:** [`darkbase-headless-preferences.md`](darkbase-headless-preferences.md).
