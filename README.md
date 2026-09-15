# unixsock-nv

A Unix-domain socket connects two processes on the same machine. It is
the `AF_UNIX` address family of the POSIX socket API, and its address is
a path in the filesystem rather than an internet address and a port.
Linux's [unix(7)](https://man7.org/linux/man-pages/man7/unix.7.html) and
the [POSIX socket calls](https://pubs.opengroup.org/onlinepubs/9699919799/functions/socket.html)
are what this package ports. It sits beside `std.net` in the standard
library: `std.net` is TCP, which reaches another machine over a port,
and this package reaches another process over a path.
[muxproto-nv](https://novo-lang.org/packages/muxproto-nv) is the control
protocol whose frames travel over the sockets this package fans out; it
is a separate package, and this one depends on nothing.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What a Unix-domain socket is

The address of a Unix-domain socket is a `sockaddr_un`, a structure
whose `sun_path` field holds the name. unix(7) gives that name three
forms. A **pathname** socket is a file the `bind` call creates in a
directory. An **abstract** socket has a name beginning with a null byte,
lives in no directory, and is reclaimed by the kernel when the last
holder closes it; it is a Linux extension. An **unnamed** socket has no
address at all, which is what an accepted connection and both halves of
a socket pair are.

Two socket types run over that address. A **stream** socket,
`SOCK_STREAM`, is a byte stream with no message boundaries, like TCP. A
**datagram** socket, `SOCK_DGRAM`, keeps message boundaries: one send is
one receive, and over `AF_UNIX` it is reliable and ordered as well. A
logger, a metrics sink and systemd's `sd_notify` use the datagram form,
because one event is one message and there is nothing to frame.

Two things a Unix-domain socket can do that no network socket can. The
kernel records the connecting process's credentials at `connect` time
and hands them to the server, which reads them with the `SO_PEERCRED`
socket option; the peer cannot lie about its user id. And a process can
send an open file descriptor to another process in an ancillary message
of type `SCM_RIGHTS`. The receiver gets a descriptor to the same open
file, at its own number, so a privileged helper can open a file the
receiver could not have opened and hand it over.

**Socket activation** is a listening socket a service did not create. A
service manager binds it, then executes the service with the descriptor
already open and two environment variables saying so: `LISTEN_FDS`
counts them and `LISTEN_PID` names the process they were meant for. A
client that connects while the service is still starting is queued by
the kernel instead of refused.

| Fact | Value | Where |
| --- | --- | --- |
| Bytes in `sun_path`, terminator included | 108 on Linux | unix(7), "Address format" |
| Bytes an abstract name spends on its leading null | 1 | unix(7), "Abstract sockets" |
| Descriptors one `SCM_RIGHTS` message may carry | 253 on Linux | unix(7), "Ancillary messages" |
| The first descriptor a service manager passes | 3 | sd_listen_fds(3) |
| The variables an activation sets | `LISTEN_FDS`, `LISTEN_PID`, `LISTEN_FDNAMES` | sd_listen_fds(3) |

## Install

```
novo pkg add unixsock-nv
```

## Example

```novo
use unixcred
use unixsock

fn main() [io, net, time]
    // The address of a daemon's socket, in the abstract namespace.
    // Nothing is created in the filesystem and nothing is left behind.
    let addr = unixsock.abstract_name("novo/example")

    // Connect, retrying every 20 milliseconds for up to two seconds.
    match unixsock.connect_deadline(addr, 2000, 20)
        Err(e) => println("no daemon there: ${e.message()}")
        Ok(s) =>
            // Ask the kernel who is on the other end. The user id is
            // the one fact a server may authorise on.
            match unixcred.peer_cred(s)
                Ok(who) => println("the peer runs as uid ${who.uid}")
                Err(c)  => println("no credentials here: ${c.message()}")

            // Send one line, then close the connection.
            match unixsock.write_str(s, "hello\n")
                Ok(n)  => println("${n} bytes sent")
                Err(w) => println("the write failed: ${w.message()}")
            unixsock.close(s)
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: unixsock-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `unixsock` | Stream sockets: the three address forms, the listener, the connection, the socket pair, reads and writes, poll and shutdown. |
| `unixdgram` | Datagram sockets, where one send is one receive and a message longer than the buffer is reported as truncated. |
| `unixcred` | The peer's credentials from `SO_PEERCRED`, and descriptor passing over `SCM_RIGHTS`. |
| `unixactivate` | The listening descriptors a service manager passed, as a value, with the `LISTEN_PID` check. |
| `unixmirror` | Several clients attached to one session: one of them drives it, the rest receive a copy of every byte. |

## How to choose an entry point

**`unixsock` is where a program starts.** It is the stream socket, and a
stream socket is what a language server, a build daemon or a privileged
helper talks over.

**`unixdgram` is for one message at a time.** Use it when every send is
one complete event and the receiver must not have to frame anything. It
has no connection, so it has no hangup: a send to a path nobody has
bound fails, and a sender learns nothing else about who is listening.

**`unixsock.socket_pair` is for two related processes.** It answers two
connected sockets with no name and no filesystem entry, so there is no
race between a bind and a connect. A parent hands one half to a child it
forked.

**`unixmirror` is for one session with several clients attached.** It
owns the fan-out and knows nothing about what the bytes mean. A program
with one client per connection does not need it.

## The rules a user needs

1. **Closing a bound listener does not remove its path.** unix(7),
   "Binding". A `bind` on a pathname address creates a filesystem entry,
   and the next `bind` to that path fails with `EADDRINUSE` even though
   nobody is listening. `close_listener` closes the socket and unlinks
   the address the listener carries, in one call. An abstract listener
   has nothing to remove.
2. **`unlink_stale` is the crash-recovery call and it is deliberate.**
   It answers `true` when it removed a socket file, `false` when there
   was nothing there, and a fault when something is still listening.
   `listen` never unlinks for you, so a second copy of a daemon cannot
   take the first one's socket.
3. **A path one byte too long binds a different socket.** unix(7),
   "Address format": `sun_path` is 108 bytes and a longer path is
   truncated, not refused. The server then reports that it is listening
   and the client gets "no such file". `addr_fits` answers before the
   bind, and `bind` refuses with `UnixPathTooLong`.
4. **An abstract name is Linux only.** unix(7), "Abstract sockets". On a
   platform without the namespace the call answers
   `UnixNoAbstractNamespace`. An abstract socket is also unreachable
   from another mount namespace, where a pathname socket can be shared.
   `addr_text` prints an abstract name with a leading `@`, which is what
   `ss -x` and `lsof` show.
5. **"Nothing to read yet" and "the peer is gone" are different
   answers.** Both are `-1` from `read(2)`. `read_byte` answers `None`
   for the first and the `UnixHangup` fault for the second, and
   `UnixReady` keeps `readable` and `hangup` in separate fields. A loop
   that treats them as one spins at full speed, or reports every idle
   moment as a disconnect. Every read here is non-blocking.
6. **A truncated datagram looks exactly like a short one.** unix(7),
   `SOCK_DGRAM`: a message longer than the receive buffer is truncated
   and the rest is discarded. Comparing lengths cannot detect it,
   because a message that exactly filled the buffer is
   indistinguishable. Read `UnixDatagramMessage.truncated`.
7. **A descriptor sent over `SCM_RIGHTS` needs at least one byte of
   payload.** unix(7), "Ancillary messages". A zero-length message with
   ancillary data is legal to write and is dropped by some kernels, so
   `send_with_rights` refuses it at the call. The descriptors that
   arrive are the receiver's own numbers, they arrive open, and
   `close_rights` is what releases them.
8. **The peer's credentials are a snapshot taken at `connect` time.**
   unix(7), `SO_PEERCRED`. The user id and group id are what a server
   authorises on. The process id is for a log line: the process may have
   exited and the number been reused, so nothing may be decided from it.
   A datagram socket has no connection, so a receiver that wants
   credentials calls `set_pass_cred` before the first message arrives.
9. **`LISTEN_PID` is a check and not a formality.** sd_listen_fds(3).
   The activation variables are inherited across `fork` and `exec`, so a
   helper a service spawns sees `LISTEN_FDS=1` naming descriptors it was
   never given. `activation()` answers `None` when `LISTEN_PID` is not
   this process, and `unset_environment` stops a child inheriting the
   problem. Read the variables once, at the top of `main`.
10. **The clients of one session are not peers.** `unixmirror` keeps one
    primary, whose input drives the session, and any number of mirrors,
    which receive every byte of output. Each client carries its own
    frame-parser cursor, because frames from several sockets arrive
    interleaved and one shared decoder joins the front of one client's
    frame to the back of another's. `drain_input` therefore answers one
    buffer per client, in the set's own order.
11. **A session with no client attached keeps running.**
    `effective_size` answers `None` there rather than a default size, so
    nothing is resized twice when a client attaches. The effective size
    is the smallest attached client's, recomputed on every attach and
    every detach. `promote_if_detached` makes the most recent mirror the
    primary when the primary leaves.
12. **`write_all` retries short writes, and `broadcast` drops a client
    that fails.** A stream socket accepts what fits in the kernel buffer
    and reports the count; `write_all` sends the whole buffer or answers
    a fault. In the fan-out the primary is written first, and a client
    whose write fails is removed from the set rather than retried.

## What is not included

- **`SOCK_SEQPACKET`.** The third socket type keeps message boundaries
  and has a connection. Linux supports it and macOS does not, so there
  is no portable behaviour to publish.
- **Peer credentials on macOS.** `LOCAL_PEERCRED` has a different
  structure and carries no process id. `peer_cred` answers
  `UnixNoPeerCred` there, because a server that read a user id of 0 out
  of a failed call would authorise everybody.
- **Asynchronous entry points.** Every read is non-blocking and every
  call returns to its caller, so an event loop composes them. `fd_of`,
  `listener_fd_of` and `fds_of` hand out the descriptors for a `poll`
  this package does not own.
- **A connection pool, a framing layer and a request-response shape.**
  All three are protocols, and this package is the transport.
- **The pseudoterminal.** `forkpty`, the terminal's raw mode, the window
  size and the child's exit status stay in the standard library's `pty`
  module and in [pty-nv](https://novo-lang.org/packages/pty-nv).
- **What the frames mean.** `UnixMirror.frame_state` is an opaque cursor
  this package threads and never interprets. The decoder is
  [muxproto-nv](https://novo-lang.org/packages/muxproto-nv)'s.

## Related packages

- `std.net` in the standard library is the TCP half: `TcpListener` and
  `TcpStream`, an address and a port, and a peer on any machine. It has
  no `AF_UNIX` surface, no peer credentials and no descriptor passing.
  Use it to reach another machine and this package to reach another
  process on this one.
- [muxproto-nv](https://novo-lang.org/packages/muxproto-nv) is the
  terminal multiplexer's control protocol, and it performs no input or
  output. It reads the frames that arrive on the sockets `unixmirror`
  holds. Take both to build a multiplexer; take this one alone for a
  language server or a privileged helper.
- [pty-nv](https://novo-lang.org/packages/pty-nv) is the pseudoterminal:
  a shell with a terminal in front of it. A multiplexer's daemon takes
  it for the panes and this package for the clients.
- [termios-nv](https://novo-lang.org/packages/termios-nv) is the local
  terminal's own settings, for the client end of the same program.
- [smtp-nv](https://novo-lang.org/packages/smtp-nv) and
  [ssh-nv](https://novo-lang.org/packages/ssh-nv) are the other host
  packages that own a socket. Both speak a protocol over TCP; this one
  speaks no protocol at all.

## Tests

```bash
novo test tests/sock_tests.nv       # 7 tests: addresses, refusals, both socket types
novo test tests/cred_tests.nv       # 6 tests: credentials, descriptors, activation
novo test tests/mirror_tests.nv     # 6 tests: the primary, the mirrors, the size
```

The constants the suite asserts are the kernel's and the service
manager's. 253 is Linux's `SCM_MAX_FD`, the most descriptors one
`sendmsg` may carry; 108 is `sun_path`; `LISTEN_FDS`, `LISTEN_PID`,
`LISTEN_FDNAMES` and the first descriptor number 3 are systemd's
`sd_listen_fds` contract, which launchd and inetd match.

A connected socket needs two processes, so the suite asserts what the
design put in a value rather than in an exchange: the three address
forms, the `sun_path` bound, and every connecting entry point against a
path nothing is listening on, which asserts the refusal. The tests
compile today and fail at run, each on the
`not implemented: unixsock-nv.<module>.<fn>` panic that is its body.
They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `unixsock.path`, `.abstract_name`, `.addr_text`, `.addr_fits`, `.sun_path_limit` | no |
| `unixsock.connect`, `.connect_deadline`, `.listen`, `.accept`, `.accept_deadline`, `.socket_pair` | no |
| `unixsock.read_byte`, `.read`, `.write_all`, `.write_str`, `.poll`, `.poll_listener`, `.shutdown` | no |
| `unixsock.close`, `.close_listener`, `.unlink_stale` | no |
| `unixsock.fd_of`, `.listener_fd_of`, `.stream_of_fd`, `.listener_of_fd` | no |
| `unixsock.UnixFault.message` | no |
| `unixdgram.open`, `.bind`, `.send_to`, `.recv_from`, `.recv_deadline` | no |
| `unixdgram.max_message_bytes`, `.close`, `.fd_of` | no |
| `unixcred.peer_cred`, `.peer_is_self`, `.set_pass_cred`, `.max_rights_per_message` | no |
| `unixcred.send_with_rights`, `.receive_with_rights`, `.close_rights` | no |
| `unixcred.UnixCredFault.message` | no |
| `unixactivate.activation`, `.unset_environment`, `.fd_at`, `.fd_named`, `.listeners` | no |
| `unixactivate.is_unix_listener`, `.is_unix_stream` | no |
| `unixmirror.headless_set`, `.accept_into`, `.remove`, `.client_count`, `.fds_of`, `.primary_of` | no |
| `unixmirror.broadcast`, `.drain_input`, `.set_size`, `.effective_size` | no |
| `unixmirror.promote_if_detached`, `.detach_primary`, `.take_reattaches` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
