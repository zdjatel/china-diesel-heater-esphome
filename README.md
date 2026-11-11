# ESPHome control for a "FDIK 12V 5KW" water (coolant) diesel heater

Read-out and start/stop from Home Assistant for the Chinese liquid (coolant) diesel parking heater
sold on AliExpress as **"FDIK 5KW 12V Water Heater Plumbing With LCD Control"** (the listing mentions
model *FDIK-w5* and compares it to Webasto/Eberspächer; it is not a Webasto/Eberspächer product).
The ESP32 sits on the single-wire bus between the heater and its 3-wire LCD switch (remote panel),
decodes every frame and can inject start/stop commands while the physical switch stays connected.

## The heater
- Manual title: *FDIK Water Parking Heater*, heater model **HE-1 Pro**, fuel-type water heater.
- Heating power per manual: 2.8 kW min … 6.0 kW max (the listing says 5 kW); rated 12.4 V, operating
  range 11.5–16 V; consumption 15–33 W without pump and fan; fuel 0.30–0.62 L/h.
- Controls: wired LCD switch (this project talks to it), plus a one-way RF remote paired from the
  switch. Switch modes: gear mode (1–6 power levels) or time mode (e.g. 40 min).
- Main harness colours from the manual (this project does not touch them): red +12 V (10–30 V),
  brown GND, orange main-board power signal, yellow K-line, and the switch lines black (indicator
  ground), gray (switch signal) and green (5 V indicator power).
- On the author's unit the 3-wire switch cable is **black / blue / red**, see *Hardware*. Check yours
  with a multimeter and a logic analyser before connecting anything: wire colours differ between batches.

> **Safety: experimental, use at your own risk.** This firmware starts a fuel-burning appliance.
> Tested on one unit only. Always supervise the first starts, keep the heater's own protections
> intact, and add an independent over-temperature shutdown. No warranty (see `LICENSE`).

## Features
- Sensors: supply voltage, vent (fan) speed %, coolant temperature, return (back-flow) temperature,
  flame temperature.
- Text sensors: state (OFF / ON / Process starting / Burn-up completed / Operating normally /
  Stopping, cooling down) and error (E-01 … E-14 with the manual's wording).
- Controls: **Heater Control Enable** switch (always OFF after boot), **Heater Start**, **Heater Stop**.
  Start is accepted only when the heater is OFF with no error; Stop is always accepted.
- Works alongside the original remote.

## Hardware
| Heater remote wire | Connect to |
|---|---|
| Black | GND |
| Blue (DATA, 3.3 V) | GPIO16 (RX) directly, and GPIO17 (TX) through a diode |
| Red (5 V) | not needed (by the manual's naming, probably the 5 V indicator-power line) |

TX diode: **cathode (stripe) to GPIO17, anode to the DATA line** (a Schottky diode is preferable).
It acts as an open-drain driver. Connecting GPIO17 directly to the line makes the heater report a
communication error. Board used: ESP32 NodeMCU-32S, ESP-IDF framework.
Your ESP32 must be powered separately; only the data line and ground are shared with the heater.

## Install
1. Copy `secrets.yaml.example` to `secrets.yaml` and fill it in.
2. Flash `heater.yaml` with ESPHome (`esphome run heater.yaml`).
3. In Home Assistant, enable **Heater Control Enable** before pressing **Heater Start**.

The heater's own error list is in `docs/protocol.md`. Per the manual, E-05 (overheat) means poor
water circulation or a too-high temperature limit, E-08 means insufficient fuel/oil pump/flame sensor,
and E-12 means pump current too high or a stuck or wrong pump. Air left in the circuit also causes
overheating; bleed the system and fill it with coolant before the first start.

First test: press Start, watch the state change `OFF → ON → Process starting…`, then press Stop
and check `Stopping, cooling down… → OFF`. Check that your system circulates coolant before a full
start; the heater trips on E-05 (overheat) or E-12 (poor circulation) otherwise.

## Protocol
See [`docs/protocol.md`](docs/protocol.md). Short version: 7500 baud 8N1, two alternating frames
(remote → heater `A6`, 26 bytes; heater → remote `6A`, 36 bytes), CRC16-Modbus, payload XOR-masked
with a per-frame key.

## Sample logs (`logs/`)
Captured with earlier diagnostic builds (the `p=[...]` lines are decoded payloads, `diff:` shows
changed bytes). `full-cycle-start-stop.log` is a complete start/run/stop driven from Home Assistant.

## Related projects
A different variant of the same family (25000 baud, unmasked 24-byte frames) is documented in
Jess--/Chinese-Heater-Temperature-Addon and wshelley's thermostat project. Your unit may match
either variant; compare raw frames before trusting this config.

## License
MIT
