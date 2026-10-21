# ha-ir-ac-comfort-thermostat

A Home Assistant pattern for running an **IR-only air conditioner** (no state feedback) as a
**single-room comfort thermostat**: pick the room you're in, set its comfort temperature,
and HA steers the AC's **fan speed and setpoint** to hold it there, optionally powering the unit off
when the room stays cool on its own and back on when it warms up (auto-off, v2.3).

It suits a central or ducted AC with one shared setpoint feeding several rooms. That setpoint can't satisfy every room at once, so the thermostat steers on whichever **one** room you're using.

## The two layers

| File | Role |
| --- | --- |
| `packages/ac_ir_control.yaml` | **Assumed-state control surface.** The AC can't report back, so helpers (`input_boolean` on/off, `input_select` mode + fan, `input_number` setpoint) *are* the state. Any helper change re-sends one full IR frame (mode+temp+fan). Power is a separate blind toggle. |
| `packages/comfort_thermostat.yaml` | **The loop.** Reads the selected room's sensor, compares to that room's target, writes the helpers. Touches power only through the optional auto-off (park / resume). |

## Control law (v2.3, every 5 min, and immediately when toggled / room or target changed)

- Only runs while the master toggle is on **and** the AC is assumed on (or parked by auto-off, below).
- Mode is forced to `cool`.
- `e = room − target`, `slope` = room trend in °C/min (from a Derivative helper, see Setup).
- The loop cycles between two phases, stored in `input_boolean.climate_manager_cooling`:
  - **IDLE → COOL** when `e ≥ +0.5`
  - **COOL → IDLE** when `e ≤ 0`, or the 10-minute forecast `e + 10·slope ≤ +0.2`. This is
    anticipation: a ducted unit keeps cooling after it stops, so it has to quit before the target.
- **COOL:** setpoint = target − 1 (target − 2 if `e ≥ 1.5`). Fan: `e ≥ 2` → high, `e ≥ 1` → med, otherwise low.
  These are the gentler values from 2026-10-21; the first version (target − 2, fan med from `e ≥ 0.5`,
  stop at `e ≤ −0.3` / forecast ≤ 0) dropped the room 0.6–0.9 °C per burst and undershot by ~0.3 °C.
- **IDLE:** setpoint 28, which keeps the compressor off in cool mode. Fan low.
- IR frames only go out when the phase or fan changes.
- Every threshold above is an `input_number.climate_manager_tune_*` helper (`start`, `stop`, `forecast_stop`,
  `horizon`, `cool_offset`, `boost_error`, `boost_offset`, `fan_med`, `fan_high`). That way you can retune without editing YAML.
  The helpers have no `initial:`, so **set them once after the first reload**: a new `input_number` starts at its `min`.
- Optional auto-room: 22:00 → Bedroom, 09:00 → Living Room. A manual pick holds until the next boundary.

### Auto-off (v2.3)

Until v2.2 the loop never toggled power: IDLE left the unit running at setpoint 28 with the fan on low,
which on a cool evening only blows the room colder. Auto-off (switch `input_boolean.climate_manager_auto_off`)
adds two transitions:

- **PARK** when the loop has been IDLE for ≥ `park_minutes` (default 30) **and** `e ≤ park_error`
  (default −0.5 °C): the room is below target on its own. The loop sets `input_boolean.climate_manager_parked`
  and sends the power toggle, turning the unit off.
- **RESUME** while parked, when `e ≥` the start threshold (`tune_start`, +0.5): power toggle on, clear
  `parked`, and go straight to COOL.
- Only a park the thermostat made is resumed. If you turn the AC off by hand it stays off. If you turn it
  on by hand while parked, the park is cleared and the normal loop takes over.
- Each park and resume writes a logbook entry (Comfort Thermostat: `parked (AC off): …` / `resumed: …`,
  with room, temperature, target and error).
- The two thresholds are `input_number.climate_manager_park_error` and `input_number.climate_manager_park_minutes`.
  They are kept apart from the `tune_*` helpers. Like those, they have no `initial:`, so a fresh install starts
  them at their `min` (−3 °C and 10 min). Set them once after the first reload, e.g.
  `input_number.set_value` with `value: -0.5` and `value: 30`.

Three template sensors (`sensor.comfort_thermostat_room_temperature`,
`sensor.comfort_thermostat_room_trend`, `sensor.comfort_thermostat_target`) expose what the loop is steering on, which makes it easy to put on a dashboard.

## Why v1 was replaced

v1 stepped the setpoint 1 °C per 10-minute tick (down when `e > +0.5`, up when `e < −0.5`, hold inside
±0.5). On a real ducted unit the room fell ~1.7 °C in 15 minutes once cooling started. By then the
walker had already wound down to the 17 °C floor, and the ±0.5 deadband held 17 while the room
overshot. Two weeks of history also showed the room barely tracks the setpoint (≈20.5–21.5 °C at
setpoints 18, 19 and 20). The setpoint behaves like an on/off lever for the compressor, so v2 uses it as one.

## Setup

1. Enable packages: `homeassistant: packages: !include_dir_named packages`.
2. Create one **Derivative** helper per room sensor: Settings → Helpers → Derivative sensor,
   source = the room temperature sensor, time window 15 min, time unit minutes, 3 decimals.
   These are UI helpers on purpose: a YAML `platform: derivative` sensor needs a full HA restart,
   while a helper is live immediately and everything else here loads with a reload.
3. Replace every `REPLACE_*` with your own entities and IR codes. That includes both sensor maps
   (temperature and trend) in the template sensors, e.g. `sensor.REPLACE_LIVING_ROOM_TREND_SENSOR`
   → your Derivative helper's entity id. If a room has no sensor yet, leave it as `''` and that room becomes a no-op.
4. `ha core check`, then `homeassistant.reload_all`. No restart is needed.
5. Set the `tune_*` and `park_*` helpers (they start at their `min`), the per-room targets, choose a room,
   and turn the toggle on. Turn on `climate_manager_auto_off` if you want auto-off.

## Caveats (learned the hard way)

- **Assumed state can drift.** If someone uses the handheld remote, HA doesn't know. The worst case is
  that HA thinks the AC is on when it's off. The loop then sits in COOL with no
  effect. If a room keeps warming while the loop is cooling, check that the AC is really on.
- **Auto-off inherits the drift.** Power is a blind IR *toggle* sent on assumed state. If the handheld
  remote has desynced HA's idea of on/off, a park turns the unit **on** and a resume turns it **off**. Leave
  auto-off disabled if the handheld remote is in regular use.
- **One frame carries everything.** A setpoint-only change re-asserts mode and fan as well.
  That's why the loop owns fan *and* setpoint, instead of having a human adjust one lever while it adjusts the other.
- A ceiling-only version (only ever lowering the setpoint) can't relieve over-cooling. The loop
  needs an explicit "stop cooling" state, which is what IDLE (setpoint 28) is.
- The real fix for drift is an IR **receiver** next to the AC that decodes the handheld remote, such as
  an ESPHome board with TX and RX. That's a hardware upgrade; the YAML pattern stays the same.

## Report

[`docs/comfort-thermostat-thermal-report-2026-10-21.pdf`](docs/comfort-thermostat-thermal-report-2026-10-21.pdf) is a plain-language write-up of the deployment this pattern came from: how it controls the AC, the v1 → v2.2 history (it predates auto-off), and ten days of indoor/outdoor temperature data (11–21 Oct 2026).

## Related

- Decoded IR protocol for an Electra central (ducted) AC, used to generate all 120 state frames:
  <https://github.com/danielrosehill/electra-central-ac-ir-codes>

MIT licensed.
