# Changelog

All notable changes to telnet-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

The first release.

### Added

- **`telnetwire`**, the bytes of a Telnet stream with no allocation: the
  constants of RFC 854 and the options, and all-or-nothing builders for `IAC
  cmd`, `IAC verb opt` and `IAC SB opt ... IAC SE`, with the doubling of 255.
- **`telnetq`**, the Q method of RFC 1143 as four functions over six states,
  so that two ends cannot answer each other's acknowledgements for ever.
- **`telnetsess`**, one end of a connection as a step: `tel_rx_byte`, which
  separates the data of the Network Virtual Terminal from commands, option
  negotiation and subnegotiation, with the CR LF and CR NUL rules and BINARY;
  `tel_data` and `tel_flush` for sending data; `tel_client`, `tel_server`,
  `tel_start` and `tel_ask` for the options; a queue of outgoing bytes; and the
  answers it makes by itself for NAWS, TERMINAL-TYPE and TERMINAL-SPEED.
- **`telnetopt`**, the subnegotiations of TERMINAL-TYPE (RFC 1091),
  TERMINAL-SPEED (RFC 1079), NAWS (RFC 1073) and NEW-ENVIRON (RFC 1572), built
  and read.
- **`telnetcom`**, a serial port over Telnet, RFC 2217: `ComPort`, the client's
  requests, and `com_handle` for both ends, with the notifications of the line
  state and the modem state.
- **`tests/embedded_probe.nv`**, a firmware program for a serial-port server,
  built for `--target=nrf52-qemu` and for `--target=nrf52840-dk
  --hci-uart=physical`. It prints `PASS: telnet-embedded` under QEMU and asks the
  arena for no bytes. The image for `nrf52-qemu` is 25,452 bytes of flash and
  520 bytes of static RAM.
- Six test suites with vectors worked out by hand, the four tables of RFC 1143,
  a random run of two ends against each other, and the opening of the BusyBox
  telnet daemon from its source; `tests/coverage.sh`, which reports 100% line
  coverage of `src/`.

### Not included

- A socket, the connection and the urgent data of the Synch.
- LINEMODE, TOGGLE-FLOW-CONTROL, X-DISPLAY-LOCATION, AUTHENTICATION and ENCRYPT.
- The commands ABORT, SUSP and xEOF of RFC 1184.
- A terminal: the data is bytes.

### Known

- A subnegotiation is held in 128 bytes and the queue of outgoing bytes in 256;
  a longer subnegotiation is reported as an error, and a reply that does not fit
  the queue is lost and counted in `tx_lost`.
