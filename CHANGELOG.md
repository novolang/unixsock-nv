# Changelog

## 0.0.1 — interface

The interface, published before anything is implemented: every `pub fn`
body is a `todo()`, and the signatures, the effect rows and the tests are
the design.

- Five modules: `unixsock` (addresses, listeners, connections, socket
  pairs), `unixdgram` (message boundaries), `unixcred` (`SO_PEERCRED`
  and `SCM_RIGHTS`), `unixactivate` (`LISTEN_FDS` as a value) and
  `unixmirror` (one primary, any number of mirrors).
- 60 public functions and two `Error` implementations, every body a
  `todo("unixsock-nv.<module>.<fn>")`.
- Three test suites, 19 tests, red on purpose.
- The half hoisted out of the standard library's `pty` module, which
  said in its own comment that the socket externs move when a second
  caller shows up.  `std.pty` keeps the pseudoterminal and is not
  wrapped; the README carries the extern-by-extern table.
