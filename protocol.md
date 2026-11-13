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
| p[15] | **mode flag of the switch**: `02` = red LED (time mode), `01` = blue LED (settings mode). It flips when the user presses △+⚙ together (that press also registers a △, so p[16] often changes by +1 in the same frame). The switch remembers it across a power cut |
| p[16] | **gear (power level) set on the switch**, 1…9 (display `L-01`…`L-09`, blue LED), ±1 per key press; the heater echoes it in `6A` p[15] = 16 + 4 × gear (about 90 ms later) |
| p[17] | **working-time timer in minutes** (decimal: 0x28 = 40 default, 0x32 = 50, 0x3C = 60; display `C-40`…`C-90`, red LED); short key press ±10 within 0…90 and wraps (90 + 10 → 00, 00 − 10 → 90; 00 is probably "no timer"); holding a key repeats each frame; the heater echoes it in `6A` p[26] |
| p[18] | 0 (key position) |
| p[23] | 0x18 constant |

## Heater frame `6A` (decoded)
| Index | Meaning |
|---|---|
| p[3] | state: 0 standby/off, 1 ON, 2 starting, 172 (0xAC) manual pump mode `P on` (lasts about 4 minutes, observed 262 s, then back to 0), 3 repeated ignition attempt ("Trying to burn it up again.."; seen with vent 59 %, flame temp 34 °C, no error), 4 burn-up completed, 5 running normally, 7 stopping/cooling, 8 OFF after a full run |
| p[4] | error: 0 standby, 1 no error, n ≥ 2 = E-(n−1). Verified for E-05, E-08, E-12. Start clears a latched error |
| p[6] | supply voltage × 0.1 V (LSB jitters) |
| p[10] | vent speed, 0 … 78 → 0 … 100 % |
| p[19] | coolant temperature °C |
| p[21] | flame temperature °C (20 cold, ~230 running) |
| p[25] | return (back-flow) temperature °C |
| p[15] | echo of the remote gear: 16 + 4 × gear |
| p[26] | in standby: echo of the remote timer in minutes (A6 p[17]). In state 5 (running): **remaining time**, counting down by 1 about every 66.5 s (e.g. 0x5A → 0x59 → 0x58 …) |
| p[7]:p[8] | probably fan speed (unverified) |
| p[32] | counter that rises together with the remaining-time countdown while running (0x2A → 0x2F over 5 steps), meaning unknown |
| p[12], p[14] | counters, unknown |

## Typical start/stop timeline (from logs/full-cycle-start-stop.log)
ON ≈ 8.5 s → starting ≈ 150–185 s → burn-up ≈ 40 s → running (warm heater, HA Start at 12:57: ON 8.5 s, starting 183.8 s, burn-up 37 s); after Stop: cooling ≈ 3 min → state 8.
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

## Buttons and settings
The switch keeps all settings itself and just sends them in every `A6` frame; the heater only echoes them in `6A`.
There is no separate "key pressed" message.

| Mode (`A6` p[15]) | LED | Display | ▲ / ▼ change | Field |
|---|---|---|---|---|
| `02` | red | `H-<timer>` (OK cycles: coolant °C → `H-<timer>` → supply V → return °C) | gear, ±1, **silently**: the display stays `H-40` | `A6` p[16] (echo `6A` p[15] = 16 + 4 × gear) |
| `02` | red | `C-<timer>` after *settings* | timer, ±10 min, wraps | `A6` p[17] (echo `6A` p[26]) |
| `01` | blue | `L-01`…`L-09` | gear, ±1 | `A6` p[16] |
| `01` | blue | `C-<timer>` (⚙ toggles between L- and C-) | timer, ±10 min, wraps | `A6` p[17] |

△+⚙ pressed together switches between the red and blue mode.

Power cut to the heater/switch (12 V removed, seen as a burst of 00/FF garbage on the bus and one `6A` frame with ~6.9 V):
- the mode flag (`A6` p[15]) is kept, so the switch comes back in the same colour;
- the gear (`A6` p[16]) is kept, whichever mode it was edited in;
- the timer (`A6` p[17]) always goes back to 40 minutes, also in the blue mode (e.g. 70 → 40 right after the burst).
Whether pressing OK would store a timer in the blue mode is not tested.

Sliders in the firmware: *Heater Gear* and *Heater Timer* are only sent with Start (the remote keeps its own values afterwards;
whether the heater keeps the injected ones is not verified).

## Notes from button tests
- `A6` p[23] shows one-frame changes (0x18 → 0x5C with key 0x44, 0x18 → 0x98 with key 0x7F) that repeat with identical raw bytes every ~15 s. They are not button presses; ignore p[23].
- In manual pump mode the heater reports state 172 (0xAC) and `6A` p[16] counts down about one step per 9.5 s.
- A power cut to the heater shows up as a short burst of 00/FF garbage on the bus, one odd `6A` frame (supply ~6.9 V, flame temperature ~53 °C) and a reset of the switch timer (`A6` p[17]) to 40 minutes within ~3 s. The gear (p[16]) survives a power cycle, the timer does not. Treat single-frame voltage and temperature outliers as glitches.
