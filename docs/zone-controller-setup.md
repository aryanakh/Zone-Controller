# Zone Controller — Home Assistant Setup Guide

This guide sets up per-room zoning for a ducted HVAC system using two Home
Assistant blueprints plus a set of helper entities. The firmware (ESPHome /
KC868) is separate — this is purely the Home Assistant control logic. For the
big-picture overview and architecture diagram, see the [README](../README.md).

## How it works

One upstairs unit serves four rooms through motorized dampers:

| Priority | Room | Damper(s) | Mode | Setpoints |
|:--:|------|:--:|------|-----------|
| 1 | Nursery | 1 | standard, tight band | Heat < 68 °F, Cool > 71 °F |
| 2 | Master bedroom | 2 | occupancy, cool-only | Cool > 75 °F day / > 68 °F Sleep |
| 3 | Server room | 1 | always-on, cool-only | Cool > 80 °F |
| 4 | Theater / bonus | 1 | relief valve | Coordinator-managed |
| — | Hallway | 0 | always open (structural relief) | — |

Each conditioned room runs a **Room Zone** automation. It computes a raw demand
(`heat` / `cool` / `none`), writes it to an `input_text` helper, and opens its
damper only when its demand matches the direction the unit is delivering. The
single **Coordinator** automation reads those demand helpers *in priority order*,
sets the unit's direction to the highest-priority calling room, turns the unit
on/off, drives the theater damper as a relief valve, and handles home/away.

Because a single air handler is hot **or** cold at any moment, priority decides
which room "wins" the unit when demands conflict; lower-priority rooms wanting the
opposite direction wait with their dampers closed.

> **Damper polarity is inverted / normally-open:** `off` = damper **OPEN**,
> `on` = damper **CLOSED**. The blueprints account for this; a power loss springs
> every damper open (failsafe). You only point them at the correct `switch`
> entities.

## Prerequisites

You need these entities to already exist in Home Assistant:

- **Room temperature sensors** — one per room (`sensor.*`, device_class `temperature`).
- **Master bedroom occupancy** — a `binary_sensor.*` that is `on` when occupied.
- **Damper switches** — one `switch.*` per damper (5 total; master bedroom has 2).
- **Main HVAC** — a `climate.*` entity that supports `cool`, `heat`, `heat_cool`, `off`.

## 1. Install the helper package

Copy [`packages/zone_controller.yaml`](../packages/zone_controller.yaml) into
your HA config at `<config>/packages/zone_controller.yaml`. Enable packages in
`configuration.yaml` if you haven't already:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Edit the package and replace every `# TODO` (the `zone:` latitude/longitude, and
confirm the setpoint defaults). Then **restart Home Assistant**. This creates:

- Setpoints: `input_number.nursery_heat_sp` / `nursery_cool_sp`,
  `server_cool_sp`, `master_cool_sp` / `master_sleep_cool_sp`
- Enable toggles: `input_boolean.nursery_enabled`, `server_enabled`, `master_enabled`
- Demand helpers: `input_text.nursery_demand`, `master_demand`, `server_demand`
- `input_boolean.zc_sleep_mode`, `input_boolean.zc_home_occupied`
- `input_datetime.zc_last_changeover`
- `zone.home_wide` (~5 mi)

## 2. Import the blueprints

**Settings → Automations & Scenes → Blueprints → Import Blueprint**, then import
both (or copy into `<config>/blueprints/automation/zone_controller/`):

- `blueprints/automation/zone_controller/room_zone.yaml`
- `blueprints/automation/zone_controller/zone_coordinator.yaml`

## 3. Create the room automations

Create **one automation per conditioned room** from the **Room Zone** blueprint.

### Nursery (priority 1 — standard, tight band)
- Room enable toggle: `input_boolean.nursery_enabled`
- Demand helper: `input_text.nursery_demand`
- Room temperature sensor: `sensor.nursery_temperature`
- Damper switch(es): `switch.damper_nursery`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.nursery_cool_sp` · Enable cooling: **on**
- Heat setpoint: `input_number.nursery_heat_sp` · Enable heating: **on**
- Hysteresis: **0.3** · Zone mode: **Standard**

### Master bedroom (priority 2 — occupancy, cool-only, sleep, 2 dampers)
- Room enable toggle: `input_boolean.master_enabled`
- Demand helper: `input_text.master_demand`
- Room temperature sensor: `sensor.master_bedroom_temperature`
- Damper switch(es): `switch.damper_master_bedroom_1`, `switch.damper_master_bedroom_2`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.master_cool_sp` · Enable cooling: **on**
- Heat setpoint: *(leave unset)* · Enable heating: **off**
- Zone mode: **Occupancy** · Occupancy sensor: `binary_sensor.master_bedroom_occupancy`
- Sleep mode boolean: `input_boolean.zc_sleep_mode`
- Sleep cool setpoint: `input_number.master_sleep_cool_sp`
- **Scheduled activation** (pre-cool for sleep):
  - Active after: **20:00:00** · Active until: **07:00:00**
  - Schedule requires house occupied: `input_boolean.zc_home_occupied`

  This makes the master bedroom start cooling around 8pm whenever the house is
  occupied, on top of its occupancy behaviour (the window runs overnight to
  07:00). It becomes active if the room is occupied **or** it's after 8pm and
  someone is home. Latency is up to the ~5 min room resync, hence "around" 8pm.

### Server room (priority 3 — always on, cool-only)
- Room enable toggle: `input_boolean.server_enabled`
- Demand helper: `input_text.server_demand`
- Room temperature sensor: `sensor.server_room_temperature`
- Damper switch(es): `switch.damper_server_room`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.server_cool_sp` · Enable cooling: **on**
- Heat setpoint: *(leave unset)* · Enable heating: **off**
- Zone mode: **Always on**

> The theater / bonus room does **not** get a Room Zone automation — its damper
> is managed only by the coordinator.

## 4. Create the coordinator automation

Create **one** automation from the **Coordinator** blueprint:

- Main HVAC climate entity: `climate.upstairs_hvac`
- **Room demand helpers, HIGHEST PRIORITY FIRST** — add in this order:
  `input_text.nursery_demand`, `input_text.master_demand`,
  `input_text.server_demand`. **Order = priority.**
- Room damper switches (all non-relief): `switch.damper_nursery`,
  `switch.damper_master_bedroom_1`, `switch.damper_master_bedroom_2`,
  `switch.damper_server_room`
- Relief valve damper switch: `switch.damper_theater`
- Minimum changeover time: **10** (minutes)
- Last-changeover helper: `input_datetime.zc_last_changeover`
- Home occupied toggle: `input_boolean.zc_home_occupied`
- Wide presence zone: `zone.home_wide`
- Away action: **Set eco preset** *(keeps the unit running so the server room
  stays cooled while the house is empty)*
- Away preset name / Home preset name: match your unit's presets (defaults
  `eco` / `none`).

> Choose **Turn the unit off** for the away action only if you have no room that
> must be conditioned regardless of occupancy — it stops the server room too.

## 5. Verify

1. **Demand + damper.** Set `input_number.nursery_cool_sp` below the nursery
   temperature. `input_text.nursery_demand` should become `cool`; with the unit
   cooling, `switch.damper_nursery` goes **off** (open). Raise the setpoint back
   and demand returns to `none`, damper **on** (closed).
2. **Enable/disable.** Turn off `input_boolean.nursery_enabled`:
   `input_text.nursery_demand` should go `none` and `switch.damper_nursery`
   **on** (closed), and the nursery drops out of arbitration.
3. **Priority conflict.** Make the nursery call for heat while the master calls
   for cooling (adjust setpoints). The unit should go/stay in **heat** for the
   nursery; the master's dampers stay **on** (closed) until the nursery is
   satisfied — subject to the changeover timer.
4. **Anti-short-cycle.** Watch `input_datetime.zc_last_changeover` — after a
   direction change, the unit won't flip heat↔cool again until
   `min_changeover` minutes pass (unless no room is still calling the current
   direction).
5. **Relief valve.** Force every room damper closed (each switch `on`):
   `switch.damper_theater` goes **off** (open). Open any room damper and the
   theater damper goes **on** (closed).
6. **Main on/off.** With a room calling, the unit switches on in the winning
   direction. With every room satisfied/disabled, it switches off.
7. **Sleep mode.** Turn on `input_boolean.zc_sleep_mode`; the master bedroom uses
   the 68 °F sleep cool setpoint.
8. **Away.** Turn off `input_boolean.zc_home_occupied` with nobody in
   `zone.home_wide`; the configured away action fires (eco preset by default).

### Template sanity check

In **Developer Tools → Template**, paste (substituting your entity IDs) to
confirm the coordinator's core expressions:

```jinja
{% set zone_demands = ['input_text.nursery_demand','input_text.master_demand','input_text.server_demand'] %}
{% set zone_dampers = ['switch.damper_nursery','switch.damper_master_bedroom_1','switch.damper_master_bedroom_2','switch.damper_server_room'] %}
winner_dir:        {% set ns = namespace(dir='none') %}{% for d in zone_demands %}{% if ns.dir=='none' and states(d) in ['heat','cool'] %}{% set ns.dir=states(d) %}{% endif %}{% endfor %}{{ ns.dir }}
all_others_closed: {{ zone_dampers | reject('is_state','off') | list | count == (zone_dampers | count) }}
```

## Notes & limitations

- **Rooms publish demand; the Coordinator owns direction.** Because arbitration
  requires it, the coordinator fully controls the unit's `hvac_mode` — a manual
  mode change will be overridden.
- Keep each room's **heat setpoint at least ~2° below its cool setpoint** so a
  room can't be cooled into needing heat (or vice-versa) and thrash.
- A room whose demand is the **opposite** of the current unit direction waits with
  its damper closed until it wins arbitration or the direction is free.
- Optional inputs (occupancy, sleep, presence zone) are re-evaluated on the
  periodic resync (coordinator every 2 min, rooms every 5 min), so worst-case
  latency for those is a couple of minutes — fine for HVAC. Enable toggles and
  demand changes are instant (they are triggers).
