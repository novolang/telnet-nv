# telnet-nv

Telnet, [RFC 854](https://www.rfc-editor.org/rfc/rfc854), is a byte stream with
a few bytes of control mixed in. Most of it is the data of a terminal. The byte
255 introduces a command, and the commands are how the two ends agree on what
the stream means: whether the server echoes, whether the bytes are binary, how
big the window is, and, for a serial port offered over the network,
[RFC 2217](https://www.rfc-editor.org/rfc/rfc2217), what the port's baud rate,
data size and parity are. This package brings that to novo-lang as a session a
caller drives over bytes it moves itself, for the client end or the server end.

## What it is

A device that offers its UART over a network link is a Telnet server with
COM-PORT-CONTROL: the program on the other end opens a connection, sets
115200 baud and 7 data bits, and the bytes of the port travel as Telnet data.
The device needs four things from a library: to separate the data from the
control, to answer the control without getting into a loop with the other end,
to turn the requests for a baud rate into settings of its UART, and to send the
UART's bytes back with the rules of the network terminal applied. This package
is those four, and the options a terminal client and server have always
traded: ECHO, SUPPRESS-GO-AHEAD, BINARY, END-OF-RECORD, NAWS (the window size),
TERMINAL-TYPE, TERMINAL-SPEED and NEW-ENVIRON.

The package has no socket and no clock. A program that uses it reads a byte from
its connection and gives it to `tel_rx_byte`, writes the bytes `tel_tx_next`
hands back, and passes the data it wants sent through `tel_data`. **Nothing is
allocated.** A session is a `@value` of 816 bytes. The whole package builds for
a Cortex-M with no heap.

**The negotiation cannot loop.** RFC 855 says what WILL, WONT, DO and DONT
mean and little about what happens when both ends send them at once, which is
how a server and a client can answer each other's acknowledgements for as long
as the connection lasts. The rules of RFC 1143, the Q method, give each side of
each option six states and a rule for every event, and an acknowledgement is
never answered. `telnetq` is those rules alone. Its tests run two ends against
each other with random requests and require every exchange to end, in agreement.

## Install

```
novo pkg add telnet-nv
```

## Example

A client is given the opening of the BusyBox telnet daemon, which asks the
client to echo and to report its window, and says that it will echo and
suppress go-ahead itself. The client answers: it does not echo (`WONT ECHO`),
it will report the window (`WILL NAWS`, followed by the window, 80 by 24), and
it asks the server to do what it offered (`DO ECHO`, `DO SGA`). A line of text
then goes out as the network terminal wants it.

```novo
use telnetsess
use telnetwire

fn main() [io]
    var c = telnetsess.tel_client()
    // IAC DO ECHO, IAC DO NAWS, IAC WILL ECHO, IAC WILL SGA.
    for b in [255, 253, 1, 255, 253, 31, 255, 251, 1, 255, 251, 3]
        let _ = telnetsess.tel_rx_byte(var c, b)
    var n = 0
    var last = 0
    for _i in 0..40
        let b = telnetsess.tel_tx_next(var c)
        if b >= 0
            n = n + 1
            last = b
    println("${n} ${last}")   // 21 3

    // "ok", a lone carriage return, "!", then a carriage return and a line feed.
    var out: telnetwire.TelBuf = Vec.new()
    for b in [111, 107, 13, 33, 13, 10]
        let _ = telnetsess.tel_data(var c, b, var out)
    println("${out.len()} ${out[3]}")   // 7 0
```

Build and test with `novo pkg build` and `novo test tests`.

## What the package contains

| Module | Contents |
| --- | --- |
| `telnetsess` | The session, `tel_client` and `tel_server`: the stream parser `tel_rx_byte` (data, commands, options, subnegotiation), the queue of bytes to send, `tel_data` for the data of the network terminal, `tel_ask` and `tel_start` for requests, and the answers it makes by itself for NAWS, TERMINAL-TYPE and TERMINAL-SPEED. |
| `telnetq` | The Q method of RFC 1143: six states, four events, and the reply to send. |
| `telnetwire` | The constants of RFC 854 and the options, and the builders for `IAC cmd`, `IAC verb opt` and `IAC SB ... IAC SE`, with the doubling of 255. |
| `telnetopt` | The subnegotiations of TERMINAL-TYPE, TERMINAL-SPEED, NAWS and NEW-ENVIRON, built and read. |
| `telnetcom` | A serial port over Telnet, RFC 2217: `ComPort`, the client's requests, and `com_handle`, which applies a request to a server's port or takes an answer into a client's mirror. |

Every `pub` declaration is documented where it is declared, and
[the package's page](https://novo-lang.org/packages/telnet-nv) is that
reference.

## The rules a user needs

1. **The caller moves the bytes.** Nothing here reads a socket. `tel_rx_byte`
   answers one event per byte: nothing, a data byte (`s.data`), a command
   (`s.cmd`), an option that came on or went off (`s.opt` and `s.side`), a
   complete subnegotiation (`s.sub`, of the option `s.sb_opt`), or an error.
2. **Empty the queue after every event.** The replies to a negotiation, and the
   subnegotiations the session answers for itself, go into a queue of 256
   bytes. Call `tel_tx_next` (a byte, or -1) or `tel_tx_drain` until nothing
   is left, after every call of `tel_rx_byte` that answered anything but nothing
   or data. A reply that does not fit is lost, and `tx_lost` counts it.
3. **Data follows the network terminal.** End of line is CR LF, and a lone CR is
   CR NUL. Received, CR NUL is delivered as the CR and the NUL is dropped. Sent,
   `tel_data` puts a NUL after a CR that no LF follows, and `tel_flush` writes
   the NUL a trailing CR is owed. A data byte of 255 is IAC IAC both ways. With
   BINARY on for a direction that direction carries bytes as they are.
4. **Each end agrees to a set of options.** `tel_client` and `tel_server` set
   which options this end performs (`local_ok`) and which it lets the peer
   perform (`remote_ok`). Anything else is refused with WONT or DONT, and so is an
   option the package does not know. A request that repeats what is already
   settled is not answered.
5. **A server opens, a client waits.** `tel_start` sends a server's offers: WILL
   ECHO, WILL SGA, and a request for NAWS, TERMINAL-TYPE and TERMINAL-SPEED.
   `tel_ask` makes any request of this end's own.
6. **Some subnegotiations are answered by the session.** When NAWS comes on for
   this end it sends the window set with `tel_set_window`. When the peer's
   TERMINAL-TYPE or TERMINAL-SPEED comes on it asks, and it stores the answer in
   `peer_ttype`, `peer_tx` and `peer_rx`. The peer's window is `cols` and `rows`.
   Everything else, NEW-ENVIRON and COM-PORT-CONTROL among them, is handed to the
   caller as `TEL_EV_SUBNEG`.
7. **A subnegotiation is at most 128 bytes.** A longer one, or one cut short by an
   IAC that is neither SE nor IAC, is reported as `TEL_EV_SB_ERROR` and dropped.
8. **A serial port is a `ComPort` and a handler.** A server gives each COM-PORT
   subnegotiation to `com_handle` with its port; the call applies the request,
   writes the answer (the code plus 100 and the value now in force) into a
   buffer, and answers a mask of `COM_CHG_` bits saying what to set on the UART.
   A request for 0 asks for the value in force and changes nothing. A client gives
   the answers to `com_handle` with its mirror. `com_notify_linestate` and
   `com_notify_modemstate` report a change when the client's mask asks for it.

## Running on a microcontroller

All five modules are `@tier(embedded)` throughout and allocate nothing, and the
package declares no effects. `tests/embedded_probe.nv` is a firmware program
for a serial-port server. It plays both ends from the same package: a client
that answers the BusyBox opening, a server and a client that negotiate ECHO,
SUPPRESS-GO-AHEAD, NAWS, TERMINAL-TYPE and TERMINAL-SPEED, a line of data with
a lone CR and a byte of 255, a client that asks for 115200 baud and 7 data bits
so that the server's port and the client's mirror change, and a modem-state
notification; and it checks that the arena was asked for no bytes. Then it prints
`PASS: telnet-embedded` and runs the bridge: the Telnet stream arrives on the
UART, data bytes are sent back, and COM-PORT requests change the port. It is
compiled for a Cortex-M as part of this package's checks, so the claim is built
rather than asserted. It builds for `nrf52-qemu`, where it prints the verdict,
and for `nrf52840-dk` with `--hci-uart=physical`.

The image for `nrf52-qemu` is 25,452 bytes of flash and 520 bytes of static RAM,
including the runtime and the board's startup code; the program's own code is
12,508 bytes of that flash. For `nrf52840-dk` it is 26,484 bytes of flash. A
session is 816 bytes, a `ComPort` 104, and a buffer of outgoing bytes 264.

The device effects are the caller's. The package reads no register and no clock,
so a call costs what its caller's transport costs. What `com_handle` says to set
on the UART is the caller's to set.

## What is not included

- **A socket,** the connection, and the urgent data of the Synch of RFC 854: the
  Data Mark is reported as a command and the caller decides what the urgent
  pointer meant.
- **The line-at-a-time modes and the other options:** LINEMODE (RFC 1184),
  TOGGLE-FLOW-CONTROL (RFC 1372) and X-DISPLAY-LOCATION (RFC 1096).
- **Authentication and encryption,** AUTHENTICATION (RFC 2941), ENCRYPT (RFC 2946)
  and TLS. A connection that needs them runs Telnet over TLS, which is the
  caller's.
- **The commands ABORT, SUSP and xEOF** of RFC 1184 (236 to 238).
- **Options the package does not name.** An offer for one is refused. The state of
  the nine named options is kept; a program with another option to carry does not
  have a place to keep it.
- **A terminal.** The data is bytes. Interpreting `ESC [` is not here.
- **The flow control of the serial line itself.** `com_handle` records the
  settings and the break, DTR and RTS requests; setting the pins is the caller's.

## Related packages

- [slip-nv](https://novo-lang.org/packages/slip-nv) and
  [ppp-nv](https://novo-lang.org/packages/ppp-nv) are the serial-line framings
  for IP, which a Telnet connection may run over.

## Tests

The suite is six files under `tests/`. What they assert comes from the RFCs
rather than from an implementation.

`tests/telnetwire_tests.nv` holds the sequences to byte strings written out
from RFC 854 and RFC 855, the doubling of 255, and the all-or-nothing rule at
every edge of the buffer.

`tests/telnetq_tests.nv` holds the four transition tables of RFC 1143 cell by
cell, typed from the RFC, then connects two ends and gives them 400 sets of
random requests, requiring every one to end in fewer than 60 messages with both
ends agreeing, and runs the classic case, both ends offering ECHO at once.

`tests/telnetopt_tests.nv` holds the subnegotiations of TERMINAL-TYPE,
TERMINAL-SPEED, NAWS and NEW-ENVIRON to byte strings, the readers for each, and
the walk over a NEW-ENVIRON list with its escapes.

`tests/telnetsess_tests.nv` holds the parser (the CR rules, the doubled IAC,
commands, an abandoned or oversized subnegotiation), the data the session sends,
the options it agrees to and refuses, the queue, and a server and a client run
against each other. The opening of the BusyBox telnet daemon is the array
`iacs_to_send` in its `networking/telnetd.c`
(<https://raw.githubusercontent.com/mirror/busybox/master/networking/telnetd.c>),
`FF FD 01 FF FD 1F FF FB 01 FF FB 03`. That is a printed array in the source and
not a capture, and the answers to it are worked out from RFC 1143.

`tests/telnetcom_tests.nv` holds RFC 2217 to byte strings: every request and
its answer, the notifications and their masks, the values that ask and change
nothing, and a request carried through the parser into a server's port and the
answer back into a client's mirror.

`tests/embedded_probe.nv` is the microcontroller program described above.

`tests/coverage.sh` merges the line coverage of the suites over `src/` and
prints it; it reads 100%.

## Licence

Apache-2.0. See `LICENSE`.
