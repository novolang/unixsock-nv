# unixsock-nv

Unix-domain sockets as a package: the rendezvous, the listener, the
connection, who the kernel says is on the other end, and the fan-out a
multiplexer's daemon runs on.

**Status: NOT IMPLEMENTED — interface only.**  Every `pub fn` body is a
`todo()`, so the signatures, the effect rows and the tests are published
and nothing is implemented.  The first implementation is the `0.1.0`
published over this.

## What this is

The standard library's `pty` module declares nine `novo_unix_*` externs
under a comment that says what they are doing there:

> Riding the `std.pty` conditional-link cascade: any user that triggers
> `novo_pty_` auto-pulls these too.  Hoist into `std.unix.*` when a
> second non-pty caller shows up.

This package is that caller.  An editor talking to a language server, a
service that was socket-activated, a build daemon, a privileged helper
handing a descriptor to an unprivileged process — none of them want a
pseudoterminal, and today all of them have to trigger one to get a Unix
socket.

Five modules, and a reader should know which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **sockets** | `unixsock` | anything. Start here |
| the **messages** | `unixdgram` | one send is one receive |
| the **peer** | `unixcred` | you need to know who, or to hand over a descriptor |
| the **inherited listener** | `unixactivate` | systemd, launchd or inetd bound it for you |
| the **fan-out** | `unixmirror` | several clients are attached to one session |

Sixty public functions and two `Error` implementations, every body a
`todo()`.

## The load-bearing interface

```novo norun:pseudo
pub struct UnixMirrorSet
    clients: [UnixMirror]
    headless: Bool
    reattaches: Int
```

**The clients attached to one session are not peers, and the set is what
says so.**  A multiplexer's server has ONE primary — the client whose
keystrokes drive the session — and any number of mirrors, which see
every byte of output.  A flat list of clients cannot express that, and
"which of these is driving" is the first question the server asks on
every tick.

Three consequences fall straight out of the type, and each one is a bug
in the shape that does not have it:

- **Every client keeps its own frame-parser cursor.**  Control frames
  arrive interleaved from several sockets, so one shared decoder splices
  the first half of one client's frame onto the second half of
  another's and acts on a verb neither of them sent.
- **A detached primary is promoted, not waited for.**
  `promote_if_detached` makes the most recent mirror the driver, because
  a server that waited for a new connection leaves every remaining
  viewer watching a session nothing can type into.
- **`headless` is a state and not an absence.**  A daemon with no client
  attached is the ordinary case, and `effective_size` answers `None`
  there rather than 24 by 80 — a default would resize every shell twice
  the moment somebody attaches.

`unixmirror` owns the sockets and the fan-out and knows nothing about
what the frames mean; `UnixMirror.frame_state` is an opaque `Int` a
caller threads through muxproto-nv's decoder.  That is why this package
has no dependency on it.

## The one example that will work

```novo
use unixsock

// Attach to a daemon, waiting out the race between the launcher's fork
// and the daemon's bind.
fn attach(name: Str) -> Result<UnixStream, UnixFault> [net, time]
    unixsock.connect_deadline(unixsock.abstract_name(name), 2000, 20)

fn main() [io]
    println("a client that waited for its daemon")
```

## Adding it, and checking it

```console
$ novo pkg add unixsock-nv
$ novo pkg build
$ novo test --isolate tests/sock_tests.nv
```

The suites are **red on purpose**: every body is a `todo()`, so every
assertion reaches `not implemented: unixsock-nv.<module>.<fn>`.
Nineteen tests across three suites, all red, every failure that message.

## The layer, and why

`host`, from the plan.  Every row is one of `[net]`, `[net, time]`,
`[net, io]` or `[io]`, and there is no `core` half to split out — a
socket package with no socket in it is an empty package.  The handful of
functions here that really are arithmetic (`addr_text`, `addr_fits`,
`sun_path_limit`, `effective_size`, `fds_of`) declare `[]` inside the
host modules, which the budget permits.

`[io]` appears beside `[net]` in exactly three places, and each is a
different thing:

- **`unlink_stale`, `close_listener`, `unixdgram.close`** — removing the
  filesystem entry a `bind` created is not a socket operation, and the
  standard library's own `novo_unix_unlink` declares `[io]` for it.
- **`unixactivate.activation` and `unset_environment`** — reading and
  clearing `LISTEN_FDS`, `LISTEN_PID` and `LISTEN_FDNAMES` is an
  environment access, which SPEC § 5.1 puts in `[io]`.
- **`unixcred.peer_is_self`** — comparing the peer's uid against this
  process's own asks a question about the process, not about the socket.

## What moves out of `std.pty`, line by line

The nine `novo_unix_*` externs are the whole of it.  Nothing else in
`std.pty` is about sockets, and nothing here re-declares an extern — a
duplicate `@ffi` wrapper emits the same LLVM symbol twice and fails IR
verification, so this package sits over `std.net`'s `unix_listen` and
`unix_connect` and over the runtime's own entry points, and `std.pty`
keeps its declarations exactly as they are.

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

**What `std.pty` keeps**, and it is the majority of the module: the
pseudoterminal itself.  `novo_pty_spawn_shell` and `novo_pty_close`,
`novo_pty_read_byte` and `novo_pty_write_byte` on the master,
`novo_pty_poll` and `novo_pty_poll_many`, the host TTY's raw mode
(`set_raw_mode`, `restore_mode`, `stdin_is_tty`), the window size
(`set_winsize`, `get_winsize`, `winsize_changed`), signalling and reaping
(`kill_signal`, `child_exited`, `child_status`), `novo_pty_spawn_daemon`
and `novo_pty_redirect_io`.  It is the runtime boundary for `forkpty` and
belongs there; the two packages coexist rather than one wrapping the
other.

**And the eight `take_*` externs go to a third place.**
`novo_pty_take_pending_resize`, `take_kill_request`,
`take_capture_request`, `take_capture_index`, `take_lssession_request`,
`take_status_request`, `take_reload_request` and `take_swsession_request`
are not sockets and not pseudoterminals: they are the multiplexer's
control protocol, and they are muxproto-nv's row.  This package carries
the bytes; that one says what they mean.

## What the kernel gives you for free, and where it is named

| the fact | why it matters | where |
| --- | --- | --- |
| the peer's uid, recorded at `connect` | the one authentication a socket gives with no secret anywhere | `unixcred.peer_cred` |
| the peer's pid, which is NOT an authorisation | the process may be gone and the number reused | the same field, and the module header says so |
| a descriptor, duplicated into the receiver | a privileged helper can open what the receiver could not | `unixcred.send_with_rights` |
| a listener bound before the process started | clients queue instead of being refused during a restart | `unixactivate.activation` |
| `LISTEN_PID`, which is a check | the variables are inherited, so a helper adopts fd 3 without it | `activation()` answers `None` |
| the abstract namespace | a socket no `rm` can break and no crash can leave stale | `unixsock.abstract_name` |

## Five places a Unix socket bites, and each one has a name

- **Closing a bound listener does not remove its path.**  The next
  `bind` fails with EADDRINUSE against a socket nobody is listening on,
  and every daemon that has shipped this bug unlinks first and hopes.
  `UnixListener` carries the address so `close_listener` removes exactly
  what it created, and `unlink_stale` is the deliberate form — it
  answers a fault when something IS listening, which is the case a blind
  `unlink` turns into two daemons, one of them unreachable.
- **A path one byte too long binds a different socket.**  `sun_path` is
  108 bytes and a longer path is TRUNCATED, not refused, so the server
  reports that it is listening and the client gets "no such file".
  `unixsock.addr_fits` and `UnixPathTooLong`.
- **EAGAIN and EOF are the same `-1` in a naive wrapper.**  One says ask
  again and the other says the peer is gone; a loop that conflates them
  spins at a hundred per cent of a core or reports every idle moment as
  a disconnect.  `read_byte` answers `None` for the first and
  `UnixHangup` for the second, and `UnixReady` keeps `readable` and
  `hangup` apart for the same reason.
- **`SCM_RIGHTS` on an empty message is dropped by some kernels.**
  Legal to write, and the descriptors intermittently never arrive.
  `send_with_rights` refuses a zero-length payload at the call.
- **A truncated datagram looks exactly like a short one.**  A receiver
  cannot tell by comparing lengths, because a message that exactly
  filled the buffer is indistinguishable.
  `UnixDatagramMessage.truncated`.

## What is deliberately absent

- **`SOCK_SEQPACKET`.**  A third socket type, supported on Linux and not
  on macOS, whose contract — message boundaries AND a connection — is
  genuinely useful and has no portable story.  A row for it belongs in
  the review rather than a half-portable surface here.
- **`async`.**  Every call blocks its task, and every read is
  non-blocking, so an event loop composes them already.  The row that
  wants `[async]` is a server with a thousand connections per cell, and
  the change is an effect row and a second set of entry points rather
  than a redesign.
- **Peer credentials on macOS.**  `LOCAL_PEERCRED` has a different
  struct and no pid.  `UnixNoPeerCred` is the honest answer until
  somebody needs it, because a server that read uid 0 out of a failed
  call would authorise everybody.
- **A connection pool, a framing layer, a request/response shape.**  All
  three are protocols, and this is the transport.

## Reference

Rust's `std::os::unix::net` is the API this ports — `UnixStream`,
`UnixListener`, `UnixDatagram`, `SocketAddr` with its abstract-namespace
case, and `pair()`.  What it does not have and this does: `SO_PEERCRED`
and `SCM_RIGHTS` (which Rust leaves to `nix` and `sendfd`), socket
activation (`libsystemd`'s `sd_listen_fds`), and the mirror set, which is
novomux's own shape.  `man 7 unix` is the normative document the module
headers transcribe.

## Licence

Apache-2.0.
