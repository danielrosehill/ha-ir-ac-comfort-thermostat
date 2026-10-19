# ha-ir-ac-comfort-thermostat

A Home Assistant pattern for running an **IR-only air conditioner** (no state feedback) as a
**single-room comfort thermostat**: pick the room you're in, set its comfort temperature,
and HA steers the AC's **fan speed and setpoint** to hold it there.

It suits a central or ducted AC with one shared setpoint feeding several rooms. That setpoint can't satisfy every room at once, so the thermostat steers on whichever **one** room you're using.

## The two layers

| File | Role |
| --- | --- |
| `packages/ac_ir_control.yaml` | **Assumed-state control surface.** The AC can't report back, so helpers (`input_boolean` on/off, `input_select` mode + fan, `input_number` setpoint) *are* the state. Any helper change re-sends one full IR frame (mode+temp+fan). Power is a separate blind toggle. |
| `packages/comfort_thermostat.yaml` | **The loop.** Reads the selected room's sensor, compares to that room's target, writes the helpers. Never touches power. |

## Control law (every 10 min, and immediately when toggled / room or target changed)

- Only runs while the master toggle is on **and** the AC is assumed on.
- Mode is forced to `cool`.
- `e = room − target`
  - fan: `e ≥ 1.5` → high, `e ≥ 0.5` → med, otherwise low
  - setpoint: `e > +0.5` → −1 °C (floor 17), `e < −0.5` → +1 °C (cap 28), otherwise hold
- One step per tick is deliberate. The loop isn't aggressive enough to oscillate, and it still closes a 2 °C gap within the hour.
- Optional auto-room: 22:00 → Bedroom, 09:00 → Living Room. A manual pick holds until the next boundary.

Two template sensors (`sensor.comfort_thermostat_room_temperature`,
`sensor.comfort_thermostat_target`) expose what the loop is steering on, which makes it easy to put on a dashboard.

## Setup

1. Enable packages: `homeassistant: packages: !include_dir_named packages`.
2. Replace every `REPLACE_*` with your own entities and IR codes. Replace the sensor
   map in the template sensor too. If a room has no sensor yet, leave it as `''` and that room becomes a no-op.
3. `ha core check`, then `homeassistant.reload_all`. No restart is needed.
4. Set the per-room targets, choose a room, and turn the toggle on.

## Caveats (learned the hard way)

- **Assumed state can drift.** If someone uses the handheld remote, HA doesn't know. The worst case is
  that HA thinks the AC is on when it's off. The loop then walks the setpoint down to the floor with no
  effect. If a room keeps warming while the setpoint sits at 17, check that the AC is really on.
- **One frame carries everything.** A setpoint-only change re-asserts mode and fan as well.
  That's why the loop owns fan *and* setpoint, instead of having a human adjust one lever while it adjusts the other.
- A ceiling-only version (only ever lowering the setpoint) can't relieve over-cooling. The loop
  needs to be symmetric around the target.
- The real fix for drift is an IR **receiver** next to the AC that decodes the handheld remote, such as
  an ESPHome board with TX and RX. That's a hardware upgrade; the YAML pattern stays the same.

## Related

- Decoded IR protocol for an Electra central (ducted) AC, used to generate all 120 state frames:
  <https://github.com/danielrosehill/electra-central-ac-ir-codes>

MIT licensed.
