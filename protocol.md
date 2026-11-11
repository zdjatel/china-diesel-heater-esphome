# Bus protocol (FDIK 12V 5KW water heater, 3-wire LCD remote)

Observed by sniffing the DATA wire (3.3 V, idle high) between the heater and its LCD switch. Not an official
document; one unit tested (FDIK HE-1 Pro / "FDIK-w5").

## Physical layer
7500 baud, 8 data bits, no parity, 1 stop bit, idle high, no inversion. Half-duplex single wire.

## Timing
Cycle ≈ 310 ms: remote frame `A6` (35 ms) → ~45 ms → heater reply `6A` (48 ms) → ~180 ms idle → next `A6`.
A receiver detects frame end after ≥ 15 ms of silence.

## Frame layout
| Frame | Header | Length | Direction |
|---|---|---|---|
| `A6` | `A6` | 26 bytes | remote → heater |
| `6A` | `6A` | 36 bytes | heater → remote |

- Byte 1 = length = total bytes − 2.
- Last 2 bytes = CRC16 Modbus (poly 0xA001, init 0xFFFF) over all preceding bytes, **big-endian**.
- Bytes 2 … n−3 are XOR-masked with a key `K` that changes every frame. `K` equals the "empty" byte
  `f[18]`; further empty bytes repeat it (`A6`: f[21], f[22]; `6A`: f[20], f[22], f[23], f[27]).
  Decode: `p[i] = f[i] ^ K` for i = 2 … n−3 (indices are whole-frame indices).
- To build a frame: take a decoded `A6` payload, change `p[3]`, mask with any `K`, recompute the CRC.
  Heater accepted frames reusing the remote's last `K`.

## Remote frame `A6` (decoded)
| Index | Meaning |
|---|---|
| p[3] | command: `00` idle, `5A` start, `5C` stop. The remote holds a command for 3–4 frames; the heater reacts to the first one |
| p[16] | varies between sessions (03 / 08 / 06), meaning unknown. Hypothesis (untested): remote settings such as gear/mode; try changing them on the switch and watch the byte |
| p[18] | 0 (key position) |
| p[23] | 0x18 constant |

## Heater frame `6A` (decoded)
| Index | Meaning |
|---|---|
| p[3] | state: 0 standby/off, 1 ON, 2 starting, 4 burn-up completed, 5 running normally, 7 stopping/cooling, 8 OFF after a full run |
| p[4] | error: 0 standby, 1 no error, n ≥ 2 = E-(n−1). Verified for E-05, E-08, E-12. Start clears a latched error |
| p[6] | supply voltage × 0.1 V (LSB jitters) |
| p[10] | vent speed, 0 … 78 → 0 … 100 % |
| p[19] | coolant temperature °C |
| p[21] | flame temperature °C (20 cold, ~230 running) |
| p[25] | return (back-flow) temperature °C |
| p[7]:p[8] | probably fan speed (unverified) |
| p[12], p[14], p[31..32] | counters, unknown |

## Typical start/stop timeline (from logs/full-cycle-start-stop.log)
ON ≈ 8.5 s → starting ≈ 150 s → burn-up ≈ 40 s → running; after Stop: cooling ≈ 3 min → state 8.
Stopping during the start phase gives state 7 for about 3 s, then state 0.

## Fault codes (from the manual)
E-01 supply undervoltage · E-02 power overvoltage · E-03 ignition plug fault · E-04 oil pump failure ·
E-05 machine overheating · E-06 fan motor fault · E-07 communication failure · E-08 flame extinguished ·
E-09 flame temperature sensor faulty · E-10 ignition failure · E-11 water circulating pump fault ·
E-12 poor circulation in waterways · E-13 water temperature sensor fault · E-14 heating temperature sensor faulty.
Manual troubleshooting hints: E-05 poor circulation or too-high temperature limit; E-08 insufficient fuel, oil pump failure or flame sensor failure; E-12 pump current too high, pump stuck or wrong pump model.
Observed: E-08 when the fuel ran out; E-05 (overheat) with coolant around 125 °C (see logs/E-05-overheat.log).

## Sending frames
Send right after the heater's `6A` reply (≈ 12 ms later). After our frame the heater's reply and the
remote's next frame can arrive back-to-back with < 15 ms gap, so a receiver must split a 62-byte
`6A`+`A6` read into two frames.
