# Changelog

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

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

### Design notes

Recorded here because the 0.0.2 README no longer carries them.

**Where the effect labels come from.** Every function declares one of
`[net]`, `[net, time]`, `[net, io]`, `[io]` or `[]`.  `[io]` appears
beside `[net]` in three places and each is a different operation.
`unlink_stale`, `close_listener` and `unixdgram.close` remove a
filesystem entry, which the standard library's own `novo_unix_unlink`
declares `[io]` for.  `unixactivate.activation` and `unset_environment`
read and clear environment variables, which SPEC § 5.1 puts in `[io]`.
`unixcred.peer_is_self` compares the peer's uid against this process's
own, which asks a question about the process rather than about the
socket.  The arithmetic functions — `addr_text`, `addr_fits`,
`sun_path_limit`, `effective_size`, `fds_of` — declare `[]` inside the
host modules rather than moving to a package of their own, because a
socket package with no socket in it would be empty.

**What this package replaces in `std.pty`.** The standard library's
`pty` module declares nine `novo_unix_*` externs under a comment saying
they ride the pseudoterminal's conditional link and should move out when
a second caller appears.  This package is that caller, and it
re-declares no extern: a duplicate `@ffi` wrapper emits the same LLVM
symbol twice and fails IR verification.

| `std.pty` extern | becomes |
| --- | --- |
| `novo_unix_listen(path)` | `unixsock.listen(UnixAddr, backlog)` |
| `novo_unix_accept(listen_fd)` | `unixsock.accept(UnixListener)` — `None` rather than `-2` |
| `novo_unix_connect(path)` | `unixsock.connect(UnixAddr)`, and `connect_deadline` for the launcher race |
| `novo_unix_close(fd)` | `unixsock.close` / `close_listener`, the second unlinking what it bound |
| `novo_unix_unlink(path)` | `unixsock.unlink_stale(UnixAddr)` — and it answers whether something IS listening |
| `novo_unix_read_byte(fd)` | `unixsock.read_byte` — `None` for EAGAIN, `UnixHangup` for EOF |
| `novo_unix_write_byte(fd, b)` | folded into `unixsock.write_all` |
| `novo_unix_write_str(fd, s)` | `unixsock.write_str` |
| `novo_unix_poll(fd, ms)` | `unixsock.poll` / `poll_listener`, answering `UnixReady` rather than a bitmask |

And the mirror machinery, which is socket bookkeeping that ended up
inside a pseudoterminal module because that is where the first caller
was:

| `std.pty` extern | becomes |
| --- | --- |
| `novo_pty_try_accept_mirror` | `unixmirror.accept_into` |
| `novo_pty_mirror_count` | `unixmirror.client_count` |
| `novo_pty_drain_mirror_input` | `unixmirror.drain_input`, one buffer per client |
| `novo_pty_promote_if_detached` | `unixmirror.promote_if_detached` |
| `novo_pty_set_primary_size` | `unixmirror.set_size` |
| `novo_pty_force_detach` | `unixmirror.detach_primary` |
| `novo_pty_set_headless` | `unixmirror.headless_set` |
| `novo_pty_set_daemon_listen` | nothing: the listener is a value the caller holds |
| `novo_pty_take_reattach_count` | `unixmirror.take_reattaches` |
| `novo_pty_mirror_byte` | folded into `unixmirror.broadcast` |

`std.pty` keeps the pseudoterminal itself: `novo_pty_spawn_shell` and
`novo_pty_close`, the master's reads and writes, the polls, the host
terminal's raw mode, the window size, signalling and reaping,
`novo_pty_spawn_daemon` and `novo_pty_redirect_io`.  The eight
`novo_pty_take_*` externs are neither sockets nor pseudoterminals: they
are the multiplexer's control protocol, which is muxproto-nv's.

**The reference this interface was shaped against.** Rust's
`std::os::unix::net` — `UnixStream`, `UnixListener`, `UnixDatagram`,
`SocketAddr` with its abstract-namespace case, and `pair()`.  Added
here: `SO_PEERCRED` and `SCM_RIGHTS`, which Rust leaves to `nix` and
`sendfd`; socket activation, which is `libsystemd`'s `sd_listen_fds`;
and the mirror set.  `man 7 unix` is the normative document the module
headers transcribe.
