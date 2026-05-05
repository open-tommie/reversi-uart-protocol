# Reversi UART Protocol Specification (Draft v0.2)

[English](README.md) | [日本語](README.ja.md)

2026-05-02

Author: [tommie.jp](https://tommie.jp) ([@tommie-jp](https://github.com/tommie-jp))

- A communication protocol for connecting a CPU that implements a Reversi
  algorithm (intended for FPGA, discrete logic, etc.) to a PC over UART, so it
  can talk to PC-side software via serial.
- The spec is written so that a CPU implementer can build a conforming
  implementation by reading this document alone.

> **Note**: "Othello" is a registered trademark of MegaHouse Corporation.
> This document uses the generic term "Reversi" instead.

## Table of Contents

- [Reversi UART Protocol Specification (Draft v0.2)](#reversi-uart-protocol-specification-draft-v02)
  - [Table of Contents](#table-of-contents)
  - [1. Document Status](#1-document-status)
  - [2. Scope](#2-scope)
  - [3. Test Setup Example](#3-test-setup-example)
  - [4. Link Layer](#4-link-layer)
  - [5. Character Set and Line Endings](#5-character-set-and-line-endings)
    - [5.1 Handling Spec Violations on Receive](#51-handling-spec-violations-on-receive)
  - [6. Buffers and Timeouts](#6-buffers-and-timeouts)
  - [7. Message Formats](#7-message-formats)
    - [7.1 PC → CPU (input to CPU)](#71-pc--cpu-input-to-cpu)
    - [7.2 CPU → PC (CPU send)](#72-cpu--pc-cpu-send)
    - [7.3 Response Code Table](#73-response-code-table)
  - [8. Coordinates and Board Orientation](#8-coordinates-and-board-orientation)
  - [9. Protocol Design Guidelines](#9-protocol-design-guidelines)
  - [10. Normal Flow](#10-normal-flow)
  - [11. CPU-side State Diagram (example)](#11-cpu-side-state-diagram-example)
  - [12. State Transition Table](#12-state-transition-table)
  - [13. Error Handling and Recovery Flow](#13-error-handling-and-recovery-flow)
    - [13.1 Recovery from Board Divergence (`RS`)](#131-recovery-from-board-divergence-rs)
  - [14. Reference](#14-reference)
    - [14.1 Serial Terminal Software Hints](#141-serial-terminal-software-hints)
    - [14.2 Design Rationale: Key Decisions](#142-design-rationale-key-decisions)
      - [Uppercase Commands (§5)](#uppercase-commands-5)
      - [CR+LF Only (§5)](#crlf-only-5)
      - [CR+LF Trade-offs](#crlf-trade-offs)
      - [`+` / `-` Response Prefix (§5, §7)](#---response-prefix-5-7)
      - [`X*` Extension Namespace (§5)](#x-extension-namespace-5)
  - [15. Open Issues](#15-open-issues)
  - [16. Design Review](#16-design-review)
  - [17. Possible Future Additions](#17-possible-future-additions)

## 1. Document Status

This is **Draft v0.2**. The specification is not finalized. Comments and
feedback are welcome.

> **Breaking changes from v0.1**: introduced `+`/`-` response prefix; removed
> `PO` mnemonic (subsumed by `+PI`); removed `ER` mnemonic (subsumed by
> `-NN`); reserved `X*` extension namespace. See [CHANGELOG.md](CHANGELOG.md).

## 2. Scope

This document defines only the **byte stream flowing on UART** (the application
protocol above the link layer).

![Scope of this document](scope.png)

PlantUML source: [scope.puml](scope.puml)

| Layer | Specified | Notes |
| ----- | --------- | ----- |
| Application (message format, state machine) | ✅ This doc | |
| Link (baud rate, byte format) | ✅ This doc | |
| Physical (USB-UART converter, voltage levels, cables, connectors) | ❌ Out of scope | CPU implementer's domain |
| PC side (terminal software, etc.) | ❌ Out of scope | PC-side implementation dependent |

**Assumption**: a USB-UART converter is plugged into the PC's USB port, and
its UART lines are wired to the CPU. Bytes flow bidirectionally at the
physical level.

## 3. Test Setup Example

CPU implementers may set up their own test environment. A typical configuration
is shown below.

![Test setup example](test-setup.png)

PlantUML source: [test-setup.puml](test-setup.puml)

- Wire CPU → USB-UART converter → PC USB port as the physical chain
- Use a serial terminal on the PC (VSCode Serial Monitor / TeraTerm / minicom)
  to send commands by hand and verify the CPU's response
- Example serial-IO match program: use Python pyserial to send/receive serial
  data and auto-respond, verifying the CPU implementation

## 4. Link Layer

- **Baud rate**: choose from 9600 / 38400 / 115200 bps, etc. (specified on the
  PC side, matching the CPU)
- **Frame**: 8N1 (8 data bits, no parity, 1 stop bit)
- **Flow control**: none
- CPU implementers may pick any baud rate within their CPU's stable range

## 5. Character Set and Line Endings

- ASCII (so a human can debug with a serial monitor)
- **Line ending is CR+LF (`\r\n`, 0x0D 0x0A) only**. Both directions strictly
  use CR+LF. LF alone or CR alone is not allowed.
- **Commands are 2-character uppercase only**. Keywords like `SB`, `SW`, `MO`,
  `PA`, `BO`, `EB`, `EW`, `ED`, `VE`, `PI`, `EN`, `RE`, `RS`, `BS`, `ST`, `NC`
  are all 2 uppercase letters. Lowercase (e.g. `pi`) is a spec violation.
- **Response prefix**: the first character of a line distinguishes
  "self-initiated notification vs response" and "success vs error". The 1-char
  scheme is inspired by POP3 (RFC 1939) `+OK` / `-ERR`.
  - `+` = **success response** to the host's most recent command (e.g.
    `+VE02name`, `+PI`, `+XB clk=...`)
  - `-` = **error response** to the host's most recent command (e.g.
    `-04 bad coord`; `<NN>` is a §7.3 code)
  - (none) = **self-initiated notification** (CPU-driven / game-flow notice;
    `MO`, `BS`, `RE`, `RS`, etc.)
- **`X*` extension namespace**: 2-letter uppercase commands **starting with `X`**
  (`XA`–`XZ`, 26 slots) are reserved as a free-form extension area for CPU
  implementers. Future official RUP commands will only use `A`–`W` as the
  first letter. PC parsers conforming to RUP must treat `X*` / `+X*` / `-X*`
  as out-of-spec-but-allowed (log only; do not return `-`).
- **Coordinates such as the `MO` argument are lowercase only** (per Othello
  notational convention: `MOd3` is OK, `MOD3` is not)
  - Column index = `ch - 'a'`, row index = `ch - '1'`

### 5.1 Handling Spec Violations on Receive

On receiving a violation (lowercase command, wrong line ending, unknown
command, etc.), both host and CPU **respond with `-<NN>\r\n` or
`-<NN> <reason>\r\n`**. `<NN>` is a 2-digit code (see §7.3; use `00` Generic
when the cause cannot be identified). The sender just logs the failure per
§7.2 #13 and does **not** auto-retry (to avoid infinite loops).

Minimum FPGA implementation: returning `-00\r\n` for every violation is
spec-compliant.

`X*` (extension namespace) is **not** treated as a violation. PC parsers log
it and do not respond with `-`.

## 6. Buffers and Timeouts

| Item | Value | Notes |
| ---- | ----- | ----- |
| CPU RX buffer | **80 bytes recommended (or more)** | longest message is `BO` (68 bytes) |
| Inter-character timeout | **100 ms** | if no next byte arrives within 100 ms mid-line, reset the parser and skip until the next `\r\n` |
| `PI`/`+PI` round trip | PC waits up to 1000 ms; 3 consecutive failures = treat as disconnected | the CPU responds immediately |
| Move timeout | configurable on PC side (default 30 s) | on timeout the PC shows a warning |

## 7. Message Formats

Legend: required = ✔ / optional = blank

### 7.1 PC → CPU (input to CPU)

Two kinds: **notifications** and **queries**. Queries require a `+`/`-`
response from the CPU; notifications make a response optional.

| # | Format | Required | Name | Kind | Meaning |
| -- | ------ | -------- | ---- | ---- | ------- |
| 1 | `SB\r\n` | ✔ | START BLACK | notify | You play black (first). Make the opening move. |
| 2 | `SW\r\n` | ✔ | START WHITE | notify | You play white (second). Wait for the opponent. |
| 3 | `MOd3\r\n` | ✔ | MOVE | notify | Opponent placed at d3. |
| 4 | `PA\r\n` | ✔ | PASS | notify | Opponent passed. |
| 5 | `BO[64-char board]\r\n` | | BOARD | notify | Send the full board state (**fixed 64 chars, row-major**; empty `0`, black `1`, white `2`). Used for sync right after connection or on recovery. CPUs that don't support reset recovery may ignore. See §8. Example: `BO0000000000000000000000000002100000012000000000000000000000000000\r\n` |
| 6 | `EB\r\n` | ✔ | END BLACK | notify | Game over, black wins |
| 7 | `EW\r\n` | ✔ | END WHITE | notify | Game over, white wins |
| 8 | `ED\r\n` | ✔ | END DRAW | notify | Game over, draw |
| 9 | `VE\r\n` | ✔ | VERSION | query | Version query. The CPU responds with `+VE<NN>[<name>]\r\n` (see §7.2 #10). |
| 10 | `PI\r\n` | ✔ | PING | query | Ping. The CPU responds with `+PI\r\n` (see §7.2 #9). |
| 11 | `X*[<args>]\r\n` | | EXTENSION | query | A CPU-implementer-defined extension query. The CPU responds with `+X*[<data>]\r\n` or `-<NN>[ <reason>]\r\n`. Example: `XB\r\n` (extension benchmark request). |

**Response policy for notifications**: responses to notifications
(`SB`–`ED`) are optional. The CPU may return `+SB\r\n` etc. as ack, but the
PC silently discards it. Only spec violations require a `-<NN>` response.

**`MO` format**: directly after `MO` are **exactly 2 chars: column (`a`-`h`)
followed by row (`1`-`8`)**. No spaces or separators (e.g. `MO d3` and
`MOd-3` are invalid). Coordinates are accepted in lowercase only (`MOd3`
OK, `MOD3` not; see §5).

### 7.2 CPU → PC (CPU send)

Three kinds: **self-initiated notifications** (`MO`, `BS`, etc.; no prefix),
**success responses** (`+CMD`), and **error responses** (`-NN`).

#### 7.2.A Self-initiated notifications (CPU-driven, no prefix)

No response from the host (the host receives and updates its state).

| # | Format | Required | Name | Meaning |
| - | ------ | -------- | ---- | ------- |
| 1 | `MOd3\r\n` | ✔ | MOVE | place at d3 (CPU's turn) |
| 2 | `PA\r\n` | ✔ | PASS | pass |
| 3 | `EN\r\n` | | END | resign |
| 4 | `RE\r\n` | | READY | boot complete. The host re-sends the most recent instruction on receipt of `RE` (recovery from in-game CPU reset; recommended for 24/7 bot operation). |
| 5 | `RS\r\n` | | REQUEST SYNC | Request board resync. The CPU sends this when it detects divergence between its own board and the host's. The host then sends the current board as `BO<64-char>\r\n` and re-sends **the most recent instruction** (typically `MO?\r\n`). See §13. |
| 6 | `BS<64char>\r\n` | | BOARD STATUS | The CPU notifies its current board snapshot at any time. Format identical to `BO` (§8 row-major 64 chars, 0=empty / 1=black / 2=white). **Report-only, no side effects** (the PC uses it for cross-check, diagnostics, spectator view). Recommended to omit in minimum FPGA implementations. |
| 7 | `ST<text>\r\n` | | STATUS | Send a thinking-status update at any time. Example: `ST d=12 n=45000\r\n`. The PC uses this for display or logging; equivalent to NBoard Protocol's `status`. `<text>` is free-form ASCII printable text. |
| 8 | `NC<nodes>,<ms>\r\n` | | NODESTATS | Optional response sent right after a move with search statistics. Example: `NC125000,450\r\n` (search nodes and elapsed milliseconds). Equivalent to NBoard Protocol's `nodestats`. |

#### 7.2.B Success responses (`+` prefix; reply to host's most recent send)

Required for queries from the PC; optional for notifications.

| # | Format | Required | Replies to | Meaning |
| - | ------ | -------- | ---------- | ------- |
| 9 | `+PI\r\n` | ✔ | `PI\r\n` | Success response to PI (replaces v0.1 `PO`). |
| 10 | `+VE<NN>[<name>]\r\n` | ✔ | `VE\r\n` | Success response to VE. `<NN>` is **fixed 2-digit ASCII protocol version** (`00`-`99`, leading zero required, `+VE1xxx` is invalid). `<name>` is an identifier (ASCII printable 0x20-0x7E, 0-16 chars; **omittable** — minimum FPGA implementations may send `+VE02\r\n`). For now only `02` (Draft v0.2) is supported. Examples: `+VE02\r\n` / `+VE02MyCPU-v1\r\n` |
| 11 | `+X*[<data>]\r\n` | | `X*\r\n` | Success response to an extension query. Example: `+XB clk=1.795MHz tick=557ns\r\n` |
| 12 | `+<CMD>\r\n` | | (any notification) | Optional ack to a notification. Example: `+SB\r\n`. The PC silently discards it. |

#### 7.2.C Error responses (`-` prefix; sent on spec violation)

| # | Format | Required | Meaning |
| - | ------ | -------- | ------- |
| 13 | `-<NN>[ <reason>]\r\n` | | Response to a spec violation. `<NN>` is **fixed 2-digit ASCII code** (`00`-`99`, see §7.3). `<reason>` is an optional ASCII printable string (max 64 chars, leading-space delimited, for debugging). Minimum FPGA implementation: just `-00\r\n` (5 bytes). The receiver only logs per §5.1; no auto-retry. |

Examples:

```text
< MOz9                  PC: invalid coordinate
> -04 bad coord          CPU: error response

< pi                    PC: lowercase violation
> -02 lowercase          CPU: error response
```

### 7.3 Response Code Table

Two-digit codes for `-<NN>`. Senders use the most specific applicable code;
fall back to `-00` (Generic) when the cause cannot be pinned down.

| Code | Name | Example trigger |
| ---- | ---- | --------------- |
| `00` | Generic / Unspecified | catch-all (used by minimum FPGA when cause is unidentified) |
| `01` | Unknown command | unknown command (`AA\r\n` etc., §5.1). The `X*` namespace is excluded (see §5). |
| `02` | Lowercase command | lowercase command (`pi\r\n`, `Pi\r\n`, etc., §5) |
| `03` | Line ending violation | non-CR+LF line ending (LF alone `PI\n`, CR alone `PI\r`, etc., §5) |
| `04` | Invalid coordinate format | `MOz9`, `MO9d`, `MOD3`, etc. (§8) |
| `05` | Illegal move | move that is not legal under the current board (see [open-issues §2](検討事項.md#2-無効な着手--不正応答への対応)) |
| `06` | Invalid protocol state | a receive that §12 forbids in the current state (e.g. `SB` mid-game) |
| `07` - `29` | Reserved | for future spec versions (v0.3+) |
| `30` - `99` | User-defined | available for individual CPU implementations |

**Minimum FPGA implementation**: sending fixed `-00\r\n` is spec-compliant.
Receivers may ignore the specific code and just log "some violation occurred".

**Recommended PC implementation**: pair the code with a reason for richer
diagnostics. Example: `-04 MO expects lowercase coord, got 'MOD3'\r\n`

## 8. Coordinates and Board Orientation

Follows the standard Othello notation (World Othello Federation / WOF style).

- **Columns**: `a`-`h` (left to right)
- **Rows**: `1`-`8` (top to bottom; `a1` is the upper-left)
- **Coordinates inside MOVE commands are lowercase** (`a`-`h`). Examples:
  `MOd3`, `MOh8`, `MOa1`
- Uppercase is also accepted (see §5)
- **`BO` ordering**: row-major `a1, b1, c1, ... h1, a2, b2, ... h8` for a total
  of 64 chars (i.e. row 1 → row 8)
- **Initial position**: `d4=W(2), e4=B(1), d5=B(1), e5=W(2)` (everything else
  empty `0`)
  → `0000000000000000000000000002100000012000000000000000000000000000`

The CPU's internal board representation (`row*8+col`, `col*8+row`, bitboard,
etc.) is **up to the implementer**. This document only defines the wire
serialization order.

![Coordinate system and BO string mapping](board-coord.png)

PlantUML source: [board-coord.puml](board-coord.puml)

## 9. Protocol Design Guidelines

- **One message per line**. Stateless, so the CPU-side parser stays simple.
- **`+`/`-` response prefix carries semantics** — the first character of a line
  distinguishes "self-initiated notification vs response" and "success vs
  error" (see §5). Inspired by POP3's `+OK`/`-ERR`.
- **Responses required only for queries** — `PI`, `VE`, `X*` queries require
  a `+`/`-` response. Notifications (`SB`, `MO`, etc.) make a response optional.
- **The CPU must not echo received bytes** — this is a machine-to-machine
  protocol, and an echo mixed into the receive stream would break parsing.
  The host should still parse robustly even if echo somehow leaks.
- **No retries** — the CPU just waits silently until timeout for a response.
- **On `RE\r\n` (READY)**, the host re-sends the most recent instruction
  (absorbing boot-timing variations). Important as a recovery mechanism for
  24/7 bot operation if the CPU resets independently.
- **Commands invalid in the current state** (e.g. receiving `MOd?` while IDLE)
  are **silently dropped** (skip until `\r\n` and reset the parser). The
  implementer may optionally return `-06\r\n` (Invalid protocol state).
- **The `X*` extension namespace is not a violation** — commands starting with
  `X` are CPU-implementer territory. PC parsers log `X*` / `+X*` / `-X*` only
  and never respond with `-`.

## 10. Normal Flow

A typical conversation after the serial connection is established. `\r\n` is
omitted in the listing.

Legend:

- `>` CPU → PC
- `<` PC → CPU

```text
< PI                    PING (boot / liveness check)
> +PI                   PI success response
< VE                    VERSION query
> +VE02MyCPU-v1         VE success response (first 2 chars are the protocol version)
< XB                    extension: benchmark request (CPU-implementer defined)
> +XB clk=1.795MHz      extension success response
< SB                    START BLACK (notification; response optional)
> MOf5                  MOVE f5 (CPU's self-initiated move; Tiger opening start)
< MOd6                  MOVE d6 (opponent's move; white reply in the Tiger opening)
  ... (alternating from here. PC interleaves PI for heartbeat)
< PI
> +PI
  ...
> EN                    END (CPU's self-initiated resign)
```

**Boot detection is PI / `+PI`-driven**. The PC actively sends `PI` and the
CPU replies with `+PI`, so the design does not depend on serial-link timing
or CPU boot order (the CPU side can be implemented purely reactively).

![Normal flow sequence diagram](flow-normal.png)

PlantUML source: [flow-normal.puml](flow-normal.puml)

## 11. CPU-side State Diagram (example)

This state machine is **one example** of a CPU implementation. As long as the
protocol semantics (message format, timing) defined here are met, the CPU's
internal state representation is up to the implementer.

<!-- markdownlint-disable-next-line MD033 -->
<img src="state-external-cpu.png" alt="CPU-side state diagram (example)" width="1000" />

PlantUML source: [state-external-cpu.puml](state-external-cpu.puml)

The base behavior is purely reactive (respond to received messages). The only
unsolicited send is `RE\r\n` right after BOOT (recovery request from in-game
reset).

| State | Role |
| ----- | ---- |
| C0: BOOT | Right after power-on. Send `RE`, then transition to IDLE. |
| C1: IDLE | Connected, no game in progress (or post-game). Wait for `SB`/`SW`. If you need to remember the game-end type (`EB`/`EW`/`ED`), keep it as an internal variable. |
| C2: MY_TURN | CPU's turn. After computing a move, send one of `MOd?` / `PA` / `EN`. |
| C3: WAIT_OPP | Opponent's turn. Wait for input. |

**End-of-game behavior**:

- When the CPU sends `EN\r\n` (resign), **the PC sends no reply** and stays
  silent until the next `SB`/`SW`. The CPU may transition to C1 (IDLE)
  immediately after sending `EN`.
- On receiving a PC-determined end (`EB`/`EW`/`ED`), the CPU also transitions
  to C1 (IDLE) and waits for the next `SB`/`SW`.
- To display the result (win/loss/draw) on an LED or LCD, store it as an
  internal CPU variable rather than as a state (the protocol does not
  distinguish it from IDLE).

**Common responses (no side effects)**:

- On receiving `PI` → send `+PI` (**valid in all states except BOOT**)
- On receiving `VE` → send `+VE<NN><name>` (**valid from C1 (IDLE) onward**;
  ignored during BOOT. `<NN>` is the 2-digit protocol version)
- On receiving `X*` (extension query) → if the CPU implementer has a handler,
  reply with `+X*[<data>]`; otherwise drop silently.

> **Note (TX/RX independence)**: UART TX and RX are independent; the CPU must
> not drop received bytes while transmitting. For example, an `MOd?\r\n` may
> arrive from the PC while the CPU is still sending `+VE<NN><name>\r\n`. The
> CPU must buffer RX (interrupt / FIFO / etc.) and parse it after sending
> completes (this is not a half-duplex protocol).

**Board sync**:

- `BO...` is accepted **only in C1 (IDLE)** and overwrites the internal board.
  Then wait for `SB`/`SW` (used right after connect / on reset recovery; not
  accepted mid-game).
- **Exception**: `BO...` arriving from the host immediately after the CPU
  proactively sent `RS\r\n` is accepted even mid-game (C2 / C3) and
  overwrites the board. See §13.1.

## 12. State Transition Table

Same machine as §11, expressed as a **state × received event** matrix
(state-transition table / Mealy form). Rows are CPU states, columns are
messages received from the PC, and each cell is "action / next state".
Implementers can read off "given event X in state Y, do Z and go to state W"
at a glance.

| state＼event | `PI` | `VE` | `X*` | `SB` | `SW` | `MOd?` | `PA` | `BO...` | `EB`/`EW`/`ED` |
| ------------ | ---- | ---- | ---- | ---- | ---- | ------ | ---- | ------- | --------------- |
| C0: BOOT | —¹ | —¹ | —¹ | —¹ | —¹ | —¹ | —¹ | —¹ | —¹ |
| C1: IDLE | send `+PI` | send `+VE name` | send `+X*` or `-NN` | →C2 | →C3 | — | — | overwrite board | — |
| C2: MY_TURN² | send `+PI` | send `+VE name` | send `+X*` or `-NN` | ※ | ※ | — | ※ | — | →C1 |
| C3: WAIT_OPP | send `+PI` | send `+VE name` | send `+X*` or `-NN` | ※ | ※ | update board / →C2 | →C2 | — | →C1 |

**Legend**:

- `send X`: send `X\r\n` (stay in current state)
- `→Cn`: transition to state Cn
- `—`: silently discard (reset parser and skip to next `\r\n`; optionally
  return `-06\r\n` for Invalid protocol state)
- `※`: **Open issue (undecided)** — see §[15. Open Issues](#15-open-issues)

**Notes**:

- ¹ While in BOOT, all received bytes are discarded. After CPU init completes,
  proactively send `RE\r\n` to transition to C1 (IDLE).
- ² C2 (MY_TURN) is a temporary state during move computation. Once done, the
  CPU spontaneously sends `MOd?\r\n` / `PA\r\n` / `EN\r\n` and transitions to
  C3 / C3 / C1 respectively (a self-transition by the CPU, not driven by an
  incoming event).
- Side assignment by `SB`/`SW` at game start: receiving `SB` makes the CPU
  black (first) and transitions to C2 (MY_TURN); receiving `SW` makes it white
  (second) and transitions to C3 (WAIT_OPP).

## 13. Error Handling and Recovery Flow

If the CPU is unexpectedly reset mid-game, it proactively sends `RE\r\n` right
after boot. The host detects `RE` and **re-sends the most recent instruction
(`SB` / `MOd?` / `BO...` etc.)** to restore the position.

```text
(CPU reset mid-game)
> RE                    READY (boot complete; re-send request)
< MOd5                  PC re-sends the previous MOVE
> MOd7                  CPU's move
  ...
```

![Recovery flow sequence diagram](flow-recovery.png)

PlantUML source: [flow-recovery.puml](flow-recovery.puml)

### 13.1 Recovery from Board Divergence (`RS`)

If the CPU detects **divergence between its own board and the host's** during
a game (for example, an `MO?` from the host is illegal on the CPU's board),
the CPU proactively sends `RS\r\n` (REQUEST SYNC). The host then sends the
current board as `BO<64-char>\r\n` and re-sends **the most recent instruction**
(typically the same opponent `MO?\r\n`).

```text
(CPU determines that the host→CPU MO?\r\n is illegal on its board)
> RS                    REQUEST SYNC
< BO0000...             PC sends the current board
< MOb5                  PC re-sends the previous MOVE
  (CPU overwrites its board from BO, then processes MO)
> MOd7                  CPU's move
  ...
```

`RS` is optional. CPU implementations that delegate board management to the
host (sync via `BO` every move) need not use it. Implementations that maintain
an internal board are encouraged to use it — it provides a self-recovery path
from PC-side bugs or bit corruption.

**Retry policy (host side)**: if `RS` arrives repeatedly without the
`RS` → `BO` plus instruction re-send resolving it, the host may abort the
game after a fixed number of attempts (e.g. 3).

## 14. Reference

### 14.1 Serial Terminal Software Hints

To honor "CR+LF only" from §5, configure the terminal in CRLF mode.

| Software | Setting |
| -------- | ------- |
| **VSCode Serial Monitor** | set line ending to `CRLF` |
| **TeraTerm** | [Setup] → [Terminal] → New-line: Receive=CR+LF, Transmit=CR+LF |
| **PuTTY** | [Terminal] → turn `Implicit CR in every LF` OFF (do not modify outgoing data). Append CR+LF explicitly to outgoing lines. |
| **minicom** (Linux/Mac) | `^A U` to set new-line mode to CR+LF (turn on Add Carriage Return) |
| **Arduino IDE Serial Monitor** | choose `Both NL & CR` in the bottom-right dropdown |

The default is often LF only, so verify the setting on first connection. If
LF only is sent, the CPU returns `ER` per §5.1.

### 14.2 Design Rationale: Key Decisions

#### Uppercase Commands (§5)

- **Simpler FPGA / MCU implementation**: received bytes can be compared
  directly against the command table; no `toupper()`-equivalent circuit /
  branch is needed.
- **Readability**: in a serial-monitor display, `PI`/`VE`/`MO` etc. visibly
  stand out as commands. The distinction from coordinates `a1`-`h8` is also
  clear (commands = uppercase, coordinates = lowercase).
- **Convention from related protocols**: most Othello-family protocols
  (e.g. NBoard Protocol) use uppercase commands.
- **Typo detection**: a lowercase send returns `ER` immediately, helping
  catch mis-implementations early.

#### CR+LF Only (§5)

- **Aligned with serial terminal defaults**: TeraTerm / PuTTY / Arduino IDE
  Serial Monitor and other major tools default to CR+LF, so manual debugging
  works out of the box without changing settings.
- **Matches the raw Enter key**: on most terminals, Enter sends CRLF.
  Typing `PI`+Enter at a serial monitor produces the spec-compliant byte
  sequence directly, so first-pass debugging feels natural.
- **De-facto standard for ASCII text protocols on the internet**: from Telnet
  (RFC 854) onward, line-oriented wire formats — HTTP / SMTP / IRC / FTP
  command channel etc. — historically use CRLF. This document follows that
  tradition.
- **Affinity with the Windows toolchain**: Notepad, PowerShell, and Command
  Prompt natively handle CR+LF.
- **Consistency with Othello-family protocols**: aligns with NBoard Protocol
  and other existing Othello-engine wire formats (see §17).
- **Robust line-end detection**: the byte sequence `0x0D 0x0A` does not occur
  in plain ASCII text, so it is more resilient against false positives than a
  bare LF (and recovers cleanly after binary-noise corruption). An isolated
  CR or LF on its own is itself a §5.1 violation signal.
- **Symmetric TX/RX, single rule**: both CPU and host can implement "`\r\n`
  ends a line, anything else is a §5.1 violation" as one uniform rule. With
  LF-only, an extra "no embedded CR" exclusion clause has to be stated
  separately, which is fuzzier.

#### CR+LF Trade-offs

- **Line-end detection needs a small state machine**: an FPGA needs a
  `CR received → wait for LF` step, but it costs only a few gates.
- **One extra byte**: `BO` grows from 67 → 68 bytes; still fits the
  recommended 80-byte RX buffer.
- **`^M` shows up in logs viewed with Unix tools (`grep`/`sed`/`awk`)**:
  visual noise but no functional impact. Strip with `tr -d '\r'`.
- **Git autocrlf concerns**: when storing serial logs or expected-output
  fixtures (§3) in the repo, set `.gitattributes` to `binary` or
  `text eol=crlf` for those files to prevent line-ending conversion at
  checkout.

Overall, the gains from **toolchain alignment**, **easy hand-typed debugging**,
and **a line-end sequence that doesn't collide with ASCII text** outweigh the
modest implementation cost on the FPGA side.

#### `+` / `-` Response Prefix (§5, §7)

Introduced in v0.2. A 1-character scheme **inspired by POP3 (RFC 1939)
`+OK` / `-ERR`**.

- **30+ years of internet-standard precedent**: POP3 has used `+OK` / `-ERR`
  since 1996 as the most widely deployed mail-fetch protocol. For an FPGA
  implementer, "success is `+`, error is `-`" is unmistakable at a glance.
- **Distinguishes "self-initiated vs response" at the grammar level**: in
  v0.1, `MO` was used both for the opponent's move and the CPU's own move,
  requiring state-machine context (MY_TURN / WAIT_OPP) to disambiguate.
  In v0.2, `+MO` (response) and `MO` (self-initiated) are immediately
  distinguishable.
- **Distinguishes "success vs error" with one byte**: `+VE02name` /
  `-04 bad` lets a parser categorize replies by the prefix alone, without
  needing to know what every error code means.
- **vs. 3-digit numeric codes (SMTP/FTP style)**: numeric codes need a
  lookup table to interpret, raising cognitive load. `+` / `-` carry meaning
  in the symbol itself.
- **Minimum FPGA implementation cost**: emit a `+` or `-` byte before the
  reply. The compare/branch logic stays trivial.
- **Mnemonic reduction**: v0.1's `PO` (PI reply) and `ER<NN>` (error reply)
  collapse to `+PI` and `-NN` in v0.2 — two fewer mnemonics to memorize.

Comparison with alternatives considered during v0.2 design:

| Option | Verdict |
| ------ | ------- |
| `+` / `-` (adopted) | POP3-tested, 1 byte, symbol carries meaning |
| 1-digit numeric (`2 OK` / `4 NG`) | needs digit→meaning table, cognitive load |
| 3-digit numeric (SMTP-style `200`) | heavy on FPGA, overkill |
| `#` prefix | suggests "comment", less direct than `+`/`-` |

#### `X*` Extension Namespace (§5)

Introduced in v0.2. A reserved area so CPU-implementer-defined commands do
not collide with the RUP core.

- **Background**: during the v0.1 period, a pico2 implementation defined
  non-RUP commands like `BM` (benchmark) and `ID` (device info). The PC
  parser then misclassified them as "unknown command" and returned `ER01`,
  causing confusion.
- **Solution**: 2-letter uppercase commands **starting with `X`** (`XA`–`XZ`,
  26 slots) are reserved as a free-form area for CPU implementers. Future
  official RUP commands will only use `A`–`W` as the first letter.
- **PC parser rule**: `X*` / `+X*` / `-X*` are not §5.1 violations. Log
  only; do not affect the state machine.
- **Renaming examples**: pico2's `BM` → `XB`, `ID` → `XI` (unified to
  2 chars under the X prefix).
- **Forward compatibility**: 26 slots are plenty for hobby CPU usage. NBoard
  Protocol-compatible extensions (e.g. `SD` for search depth) can be
  implemented as `XS` etc.

Comparison with alternatives:

| Option | Verdict |
| ------ | ------- |
| `X*` namespace (adopted) | clear collision avoidance, 2-char uniform, future-proof |
| Allow any 2-char | future RUP additions could collide |
| 3-char (`XID` etc.) | asymmetric with existing RUP; readability gain marginal |
| `#` prefix | suggests "comment", overlaps with response semantics |

## 15. Open Issues

Unresolved issues during the draft phase are tracked in a separate file:
[open-issues.md (検討事項.md)](検討事項.md). Implementer feedback is welcome.

## 16. Design Review

Review notes from four perspectives (custom CPU author / FPGA designer /
Reversi player / PC-side implementer) live in a separate file:
[design-review.md (設計レビュー.md)](設計レビュー.md).

## 17. Possible Future Additions

Candidates for future additions, considering Reversi/Othello conventions and
compatibility with other protocols.

- **Add NBoard Protocol to §14 Reference**: a related Othello engine
  communication protocol. Citing it as another reference example would make
  the design rationale more convincing for CPU implementers.
- **Note Thor notation compatibility in §8**: stating that the `f5` part of
  `MOf5` matches the coordinate notation of the Thor games database (WOF
  official) would help with replay/analysis interop.
- **Add "automatic opening-name detection" to [open-issues](検討事項.md)**:
  whether to include this in scope. The likely conclusion is "PC-side
  feature" (the CPU does not need to know opening names), but explicitly
  mentioning it once avoids re-litigating the design later.
- **Add "time control is out of scope" to §9**: WOF official rules include
  e.g. 30-min sudden-death and Fischer increment, but this document only
  defines per-move timeout; whole-game time management is a PC-side / rules-
  layer concern.
- **Add "Thor-format game export" to [open-issues](検討事項.md)**: whether
  this protocol's results should be converted to Thor format and stored on
  the PC. Where to draw the line between CPU and PC.
- **★ Stronger NBoard Protocol compatibility** (details in [NBoard互換性検討.md](NBoard互換性検討.md))
  - ✅ **Already reflected**: `ST<text>\r\n` (STATUS), `NC<nodes>,<ms>\r\n`
    (NODESTATS), the `p<N>` protocol-version suffix in the `VE` reply (all
    optional CPU-side features)
  - Not yet (medium-to-high cost): `SD<n>` (search depth, can be implemented
    as the v0.2 `XS` extension), `HI<n>` (candidate-move query, e.g. `XH`),
    `BH` (Thor-format history transfer), GGF compatibility mode
