# Zone Controller — Home Assistant Setup Guide

This guide sets up per-room zoning for a ducted HVAC system using two Home
Assistant blueprints plus a small set of helper entities. The firmware
(ESPHome / KC868) is separate — this is purely the Home Assistant control logic.

## How it works

One upstairs unit serves four rooms through motorized dampers:

| Room | Damper(s) | Role | Setpoints |
|------|-----------|------|-----------|
| Nursery | 1 | Standard, tight band | Heat < 68 °F, Cool > 71 °F |
| Server room | 1 | Always on (cool only) | Cool > 80 °F |
| Master bedroom | 2 | Occupancy (cool only) | Cool > 75 °F day / > 68 °F in Sleep mode |
| Theater / bonus | 1 | Relief valve | Coordinator-managed |
| Hallway | 0 | Always open (structural relief) | — |

Each room runs an instance of the **Room Zone** blueprint, which opens its
damper when the room is out of band and closes it when satisfied. A single
**Coordinator** instance turns the main unit on/off, drives the theater damper
as a pressure-relief valve, and handles home/away.

> **Damper polarity is inverted / normally-open:** `off` = damper **OPEN**,
> `on` = damper **CLOSED**. The blueprints already account for this — a power
> loss springs every damper open (failsafe). You only need to point the
> blueprints at the correct `switch` entities.

## Prerequisites

You need these entities to already exist in Home Assistant:

- **Room temperature sensors** — one per room (`sensor.*`, device_class `temperature`).
- **Master bedroom occupancy** — a `binary_sensor.*` that is `on` when occupied.
- **Damper switches** — one `switch.*` per damper (5 total; master bedroom has 2).
- **Main HVAC** — a `climate.*` entity that supports `cool`, `heat`, `heat_cool`, `off`.

## 1. Install the helper package

Copy [`packages/zone_controller.yaml`](../packages/zone_controller.yaml) into
your HA config at `<config>/packages/zone_controller.yaml`.

Enable packages in `configuration.yaml` if you haven't already:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Edit the package and replace every `# TODO` (the `zone:` latitude/longitude, and
confirm the setpoint defaults). Then **restart Home Assistant**.

This creates:

- `input_number.nursery_heat_sp` / `nursery_cool_sp`
- `input_number.server_cool_sp`
- `input_number.master_cool_sp` / `master_sleep_cool_sp`
- `input_boolean.zc_sleep_mode`
- `input_boolean.zc_home_occupied`
- `zone.home_wide` (~5 mi)

## 2. Import the blueprints

In Home Assistant: **Settings → Automations & Scenes → Blueprints → Import
Blueprint**, and import each of these raw URLs (or copy the files into
`<config>/blueprints/automation/zone_controller/`):

- `blueprints/automation/zone_controller/room_zone.yaml`
- `blueprints/automation/zone_controller/zone_coordinator.yaml`

## 3. Create the room automations

Create **one automation per room** from the **Room Zone** blueprint.

### Nursery (standard, tight band)
- Room temperature sensor: `sensor.nursery_temperature`
- Damper switch(es): `switch.damper_nursery`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.nursery_cool_sp`
- Enable cooling: **on**
- Heat setpoint: `input_number.nursery_heat_sp`
- Enable heating: **on**
- Hysteresis: **0.3**
- Zone mode: **Standard**

### Server room (always on, cool only)
- Room temperature sensor: `sensor.server_room_temperature`
- Damper switch(es): `switch.damper_server_room`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.server_cool_sp`
- Enable cooling: **on**
- Heat setpoint: *(leave unset)*
- Enable heating: **off**
- Zone mode: **Always on**

### Master bedroom (occupancy, cool only, sleep override, 2 dampers)
- Room temperature sensor: `sensor.master_bedroom_temperature`
- Damper switch(es): `switch.damper_master_bedroom_1`, `switch.damper_master_bedroom_2`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.master_cool_sp`
- Enable cooling: **on**
- Heat setpoint: *(leave unset)*
- Enable heating: **off**
- Zone mode: **Occupancy**
- Occupancy sensor: `binary_sensor.master_bedroom_occupancy`
- Sleep mode boolean: `input_boolean.zc_sleep_mode`
- Sleep cool setpoint: `input_number.master_sleep_cool_sp`

> The theater / bonus room does **not** get a Room Zone automation — its damper
> is managed only by the coordinator below.

## 4. Create the coordinator automation

Create **one** automation from the **Coordinator** blueprint:

- Main HVAC climate entity: `climate.upstairs_hvac`
- Room damper switches (all non-relief): `switch.damper_nursery`,
  `switch.damper_server_room`, `switch.damper_master_bedroom_1`,
  `switch.damper_master_bedroom_2`
- Relief valve damper switch: `switch.damper_theater`
- Active HVAC mode: **Heat / Cool**
- Home occupied toggle: `input_boolean.zc_home_occupied`
- Wide presence zone: `zone.home_wide`
- Away action: **Set eco preset** *(keeps the unit running so the server room
  stays cooled while the house is empty)*
- Away preset name / Home preset name: match your unit's presets (defaults
  `eco` / `none`).

> Choose **Turn the unit off** for the away action only if you have no room that
> must be conditioned regardless of occupancy — it stops the server room too.

## 5. Verify

Run these checks after everything is enabled:

1. **Damper follows demand.** Temporarily set `input_number.nursery_cool_sp`
   below the current nursery temperature. Within a few seconds (or the /5 min
   resync) `switch.damper_nursery` should go **off** (open). Raise the setpoint
   back above the temperature and it should go **on** (closed).
2. **Relief valve.** Force every room damper closed (turn each switch `on`).
   `switch.damper_theater` should go **off** (open). Open any room damper (turn
   a switch `off`) and the theater damper should go **on** (closed).
3. **Main on/off.** With at least one room damper open, confirm the main unit is
   switched on to `heat_cool`. Close all room dampers and confirm it switches
   off.
4. **Sleep mode.** Turn on `input_boolean.zc_sleep_mode`; the master bedroom
   should now use the 68 °F sleep setpoint (its damper opens above 68 °F).
5. **Away.** Turn off `input_boolean.zc_home_occupied` with nobody in
   `zone.home_wide`; confirm the configured away action fires (eco preset by
   default). Turn it back on to restore.

### Template sanity check

In **Developer Tools → Template**, paste (substituting your damper entity IDs)
to confirm the coordinator's core expressions evaluate as expected:

```jinja
{% set zone_dampers = ['switch.damper_nursery','switch.damper_server_room','switch.damper_master_bedroom_1','switch.damper_master_bedroom_2'] %}
any_open:          {{ zone_dampers | select('is_state','off') | list | count > 0 }}
all_others_closed: {{ zone_dampers | reject('is_state','off') | list | count == (zone_dampers | count) }}
```

## Notes & limitations

- An **open room damper is the demand signal** — no extra demand helpers needed.
- The coordinator only switches the unit on (to the active mode) when it is
  currently `off`, so a manual mode choice while it is running is respected.
- Run the unit in **heat_cool** for correct simultaneous heating and cooling
  across rooms. In a single-direction season, a room wanting the opposite
  direction will open its damper harmlessly but won't be satisfied.
- Optional inputs (occupancy, sleep, presence zone) are re-evaluated on the
  periodic resync, so worst-case latency for those is a couple of minutes —
  fine for HVAC.
