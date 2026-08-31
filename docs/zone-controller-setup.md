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
  `server_cool_sp`, `master_cool_sp` / `master_sleep_cool_sp` /
  `master_away_cool_sp`, `theater_cool_sp` / `theater_sleep_cool_sp`
- Enable toggles: `input_boolean.nursery_enabled`, `server_enabled`,
  `master_enabled`, `theater_guest_mode`
- Demand helpers: `input_text.nursery_demand`, `master_demand`, `server_demand`,
  `theater_demand`; per-room status: `nursery_status`, `master_status`,
  `server_status`
- `input_datetime.zc_last_changeover`
- Setpoint force-run: `input_boolean.zc_setpoint_manual`,
  `input_number.zc_last_cmd_setpoint`, `input_datetime.zc_manual_until`
- Full manual override: `input_boolean.zc_bypass`
- `zone.home_wide` (~5 mi)

> The **sleep-mode** and **home-occupied** booleans are **not** created by the
> package — use your existing helpers. The steps below refer to them as
> `input_boolean.<your_sleep_mode>` and `input_boolean.<your_home_occupied>`;
> substitute your own entity IDs.

## 2. Import the blueprints

**Settings → Automations & Scenes → Blueprints → Import Blueprint**, then import
both (or copy into `<config>/blueprints/automation/zone_controller/`):

- `blueprints/automation/zone_controller/room_zone.yaml`
- `blueprints/automation/zone_controller/zone_coordinator.yaml`

## 3. Create the room automations

Create **one automation per conditioned room** from the **Room Zone** blueprint.

Each room also has an optional **Room status output** (`input_text`) it writes a
plain-English explanation to — its temperature vs its effective target, day/sleep,
whether it's yielding or disabled — so you can see *why* each room is or isn't
calling. Point each room at its own (`nursery_status`, `master_status`,
`server_status`).

Also set every room's **Full manual override / bypass** input to the shared
`input_boolean.zc_bypass` (the same one the Coordinator uses). While that toggle
is on, the room stops driving its damper so you have full manual control; off, it
resumes. (Omitted from the per-room lists below for brevity — set it on each.)

### Nursery (priority 1 — standard, tight band)
- Room enable toggle: `input_boolean.nursery_enabled`
- Demand helper: `input_text.nursery_demand`
- Room status output: `input_text.nursery_status`
- Room temperature sensor: `sensor.nursery_temperature`
- Damper switch(es): `switch.damper_nursery`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.nursery_cool_sp` · Enable cooling: **on**
- Heat setpoint: `input_number.nursery_heat_sp` · Enable heating: **on**
- Hysteresis: **0.3** (half-band on each side → the room settles within ~0.6° of
  the setpoint) · Zone mode: **Standard**
- **Priority Yield** section → *Urgent-demand output*: `input_text.nursery_urgent`
  · *Urgent margin*: **2** — the nursery publishes `cool`/`heat` here only while
  it's more than 2° past its setpoint, so lower-priority rooms yield to it **only
  in extreme cases** (e.g. 75° when it needs 70°) and stop once it recovers to
  ~72°. (Leave its own "Yield to..." unset — the nursery never yields.)

### Master bedroom (priority 2 — occupancy, cool-only, sleep, 2 dampers)
- Room enable toggle: `input_boolean.master_enabled`
- Demand helper: `input_text.master_demand`
- Room status output: `input_text.master_status`
- Room temperature sensor: `sensor.master_bedroom_temperature`
- Damper switch(es): `switch.damper_master_bedroom_1`, `switch.damper_master_bedroom_2`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.master_cool_sp` · Enable cooling: **on**
- Heat setpoint: *(leave unset)* · Enable heating: **off**
- **Priority Yield** *(optional)*: *"Yield to this higher-priority room's demand
  helper"* → `input_text.nursery_urgent`. The master then closes its damper only
  while the nursery is **urgent** (far past its target), concentrating air on the
  nursery until it recovers to ~72°, then reopens — while both keep calling. (Point
  it at `input_text.nursery_demand` instead if you want the master to yield
  *whenever* the nursery is served, not just in extreme cases; leave unset for no
  yielding at all.)
- Zone mode: **Occupancy** · Occupancy sensor: `binary_sensor.master_bedroom_occupancy`
- Sleep mode boolean: `input_boolean.<your_sleep_mode>`
- Sleep / pre-cool cool setpoint: `input_number.master_sleep_cool_sp`
- **Away Setpoint Override:** Home-occupied toggle → `input_boolean.<your_home_occupied>`
  · Away cool setpoint → `input_number.master_away_cool_sp` (**75**). While away
  (home toggle off) the master stays eligible **even though it's occupancy-gated**
  and is capped at 75° — so it can't bake to 78 while you're out, but still runs
  warmer than its day target to save power.
- **Scheduled activation** (pre-cool for sleep):
  - Active after: **20:00:00** · Active until: **07:00:00**
  - Schedule requires house occupied: `input_boolean.<your_home_occupied>`
  - Start the window earlier when this is on: `input_boolean.theater_guest_mode`
    · Early-start minutes: **60** — when the theater guest room is running (extra
    load, less airflow per room), pre-cool begins at **19:00** instead of 20:00 so
    the bedroom still reaches its sleep target on time.

  This makes the master bedroom start pre-cooling around 8pm whenever the house
  is occupied, on top of its occupancy behaviour (the window runs overnight to
  07:00). Inside the window the room automatically switches to the **68 °F**
  sleep/pre-cool setpoint — you do **not** need to flip Sleep mode; the toggle is
  just a manual override to pull the target down at any other time. So at, say,
  9pm and 74° the room will already be cooling toward 68. It becomes active if
  the room is occupied **or** it's after 8pm and someone is home. Latency is up
  to the ~5 min room resync, hence "around" 8pm.

### Server room (priority 3 — always on, cool-only)
- Room enable toggle: `input_boolean.server_enabled`
- Demand helper: `input_text.server_demand`
- Room status output: `input_text.server_status`
- Room temperature sensor: `sensor.server_room_temperature`
- Damper switch(es): `switch.damper_server_room`
- Main HVAC climate entity: `climate.upstairs_hvac`
- Cool setpoint: `input_number.server_cool_sp` · Enable cooling: **on**
- Heat setpoint: *(leave unset)* · Enable heating: **off**
- Zone mode: **Always on**
- **Priority Yield:** *leave unset.* The server must never stop getting air, so it
  should not yield to the nursery (or any room). When other rooms yield to the
  nursery and only the nursery is open, the theater relief valve provides the
  pressure path — the server does not need to.

> The theater / bonus room does **not** get a Room Zone automation — its damper
> is managed only by the coordinator.

## 4. Create the coordinator automation

Create **one** automation from the **Coordinator** blueprint:

- Main HVAC climate entity: `climate.upstairs_hvac`
- **Room demand helpers, HIGHEST PRIORITY FIRST** — add in this order:
  `input_text.nursery_demand`, `input_text.master_demand`,
  `input_text.server_demand`, and (if you use guest mode) `input_text.theater_demand`
  **last**. **Order = priority** — the theater goes last so it's the
  lowest-priority zone and yields its airflow to the rooms above it.
- Room damper switches (all non-relief): `switch.damper_nursery`,
  `switch.damper_master_bedroom_1`, `switch.damper_master_bedroom_2`,
  `switch.damper_server_room`
- Relief valve damper switch: `switch.damper_theater`
- Minimum open zones while cooling: **2** — while cooling, the theater relief
  damper opens whenever fewer than 2 room dampers are open, so the unit never runs
  through a single small zone (high static pressure). Set to **1** for the
  original behavior. Heating is unaffected; it only ever opens the theater.
- Compressor reversal cooldown: **3** (minutes)
- Last-changeover helper: `input_datetime.zc_last_changeover`
- Status / decision output *(optional but recommended)*: `input_text.zc_status`
  — the coordinator writes a one-line summary of every decision here, e.g.
  `Starting cooling - set thermostat to 72° - Home`. Add it to a dashboard
  (or just watch it in Developer Tools → States) to see exactly what the
  coordinator did and why on each run. See **Reading the status line** below.
- Full manual override / bypass *(optional)*: `input_boolean.zc_bypass` — while on,
  the coordinator touches nothing. **Use this same helper in every Room Zone too.**
- House mode selector *(optional)*: `input_select.<your_house_mode>` — when it
  reads **Away** the unit is forced into the eco preset (Eco mode); when it reads
  **Vacation** the unit is turned off. Any other value falls back to the
  presence logic below.
  - House mode "Away" value: **Away** · House mode "Vacation" value: **Vacation**
    *(match your selector's exact option labels)*
- Home occupied toggle: `input_boolean.<your_home_occupied>`
- Wide presence zone: `zone.home_wide`
- Away action: **Set eco preset**
- Away preset name / Home preset name: match your unit's presets (defaults
  `eco` / `none`).

> **Eco Away caps each room at its own away setpoint.** When nobody is home with
> the eco action (or the house-mode selector is **Away**), the eco preset is
> applied and the coordinator keeps **serving rooms** — but each room uses its
> **away cool setpoint** (see each Room Zone's *Away Setpoint Override*), which you
> set warmer than its day target to save power while still keeping it from baking
> (e.g. the master at 75). This is per-room, so an unoccupied bedroom can't drift
> to 78 the way a hallway-only band allowed. Use **Vacation** to turn the unit
> fully off instead.
- Night starts / ends at: **21:00:00** / **07:00:00** *(label-only — drives the
  `Home Night` vs `Home Day` status; the window may cross midnight)*
- Night/sleep toggle *(optional)*: `input_boolean.<your_sleep_mode>` — while on,
  the status shows `Home Night` regardless of the clock. Changes no conditioning.
- **Idle Behavior** (what to do when no room is calling):
  - Idle behavior: **Hold a heat_cool safety band** — keeps the unit in
    `heat_cool` on the band below, so the whole house stays between the limits
    (a high *and* low backstop). Requires a unit that supports a `heat_cool` /
    auto range mode. The other options are **Turn the unit off** (original) and
    **Off, but force cooling above the high limit** (cooling-only cap).
  - High limit / band high: **76** — the `heat_cool` cool target (or, in cap
    mode, the temperature above which cooling is forced).
  - Band low: **64** — the `heat_cool` heat target (band mode only).
  - A room's own demand always takes priority over the idle backstop. **Band mode
    requires Setpoint Force-Run (below) to be enabled:** while idling the band
    uses the unit's native dual-setpoint control, but when a room starts calling
    the unit switches to a single mode and inherits the band's target (e.g. the
    high) — Force-Run is what overrides that so the unit actually runs for the
    room. (Cap mode likewise needs Force-Run on to run.)

> ⚠️ **Vacation turns the whole unit off**, which also stops the always-on
> server room. If the server room must keep cooling while you travel, use
> **Away** instead of Vacation. A house-mode change is applied on the next
> safety re-sync (within ~2 min), not instantly.
- **Setpoint Force-Run** (optional but recommended — see below):
  - Manage the unit's setpoint: **on**
  - Force-run offset: **2** · Setpoint floor: **60** · Setpoint ceiling: **85**
  - Manual setpoint override toggle: `input_boolean.zc_setpoint_manual`
  - **Enable auto manual-edit grace: off** *(recommended — see note)*
  - Manual-edit grace / last-commanded / grace-until helpers: only needed if you
    turn the grace on; leave them unset otherwise.

- **Theater Guest Room** *(optional — only when a guest sleeps in the theater)*:
  - Guest room mode toggle: `input_boolean.theater_guest_mode`
  - Theater temperature sensor: `sensor.<your_theater_temp>`
  - Theater cool setpoint: `input_number.theater_cool_sp`
  - Theater heat setpoint: *(leave unset for cool-only, like the master)*
  - Theater sleep toggle / sleep cool setpoint: your sleep boolean +
    `input_number.theater_sleep_cool_sp` *(optional — for a colder night target)*
  - Theater hysteresis: **0.5**
  - Theater demand helper: `input_text.theater_demand` — **and also add this same
    helper to the "Room demand helpers" list above as the LAST entry.** That lets
    the guest room turn the unit on and take part in arbitration, and makes the
    theater the **lowest-priority zone**: it closes its damper (yields its air) to
    the nursery/master/server whenever they're being served, and only gets air
    when no higher-priority room needs that direction. It still opens as the
    relief valve when every other damper is closed.

> **The auto manual-edit grace is OFF by default — keep it off.** It tried to
> auto-detect a by-hand setpoint change and back off, but it proved too sensitive
> to thermostat quirks (stepping, clamping, °C/°F rounding) and could get stuck
> reporting "recent manual change - leaving thermostat alone." With
> **Enable auto manual-edit grace = off** (default), force-run always manages the
> setpoint and this can never happen. For deliberate manual control, use the hard
> `zc_setpoint_manual` toggle instead — that never gets stuck. (Only turn the
> grace on if you specifically want hand-edit detection and accept the risk; it
> also needs the last-commanded and grace-until helpers set.)

> **Guest room mode** turns the theater from a passive relief valve into a
> conditioned bedroom. While the toggle is on and the theater is above its cool
> setpoint it calls for cooling like any room and its damper opens to serve it;
> while off (or not calling) the theater keeps working as the pressure-relief
> valve. Because the theater is also the relief path, it needs Setpoint Force-Run
> on for the unit to actually run for it. A new guest's temperature is picked up
> on the next re-sync (within ~2 min).

> Choose **Turn the unit off** for the away action only if you have no room that
> must be conditioned regardless of occupancy — it stops the server room too.

### Why Setpoint Force-Run?

The central thermostat senses the **hallway**. If the hallway is 67 °F, a room
wants cooling at 71 °F, and the thermostat's setpoint is still 72 °F, the unit
**won't start** — the hallway is already "cool enough" as far as the thermostat
is concerned. With force-run enabled, whenever a room wins the arbitration the
coordinator pushes the unit's target a couple of degrees past the sensed hallway
temperature (e.g. cool → 65 °F) so it actually runs; the room dampers then do the
real regulation and the coordinator turns the unit off once every room is
satisfied. The target is clamped between the floor and ceiling.

**Keeping manual control.** Three levels, lightest to heaviest:
- **Setpoint only** — turn `input_boolean.zc_setpoint_manual` **on**. The
  coordinator stops touching the *setpoint* (you pick the temperature) but still
  runs on/off, mode, and the dampers. Off = hand setpoint control back.
- **Full manual override / bypass** — turn `input_boolean.zc_bypass` **on**. The
  coordinator **and every room stand down completely**: no on/off, no mode, no
  setpoint, no damper moves. You have total manual control of the thermostat and
  all dampers until you turn it off, at which point automation resumes (within
  the ~2 min re-sync). **Point every Room Zone's *Full manual override* input and
  the Coordinator's at this same `zc_bypass` toggle.** The status lines read
  `Manual override ON ...` while it's active.
- **Soft (only if you enabled the off-by-default grace)** — change the setpoint by
  hand and the coordinator leaves it alone for the grace period. Off by default;
  prefer the toggles above.

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
   satisfied. Priority is never yielded to the lower-priority master — the only
   thing that can briefly delay the switch to heat is the compressor cooldown.
   - **Priority yield (extreme only).** With the master's *Priority Yield* pointed
     at `input_text.nursery_urgent`: push the nursery **more than 2° past** its
     cool setpoint. `nursery_urgent` → `cool`, and while the nursery is served the
     master's dampers go **on** (closed) so airflow focuses on the nursery. As the
     nursery recovers to within ~2° (`nursery_urgent` → `none`), the master's
     dampers reopen and both share — even though the nursery is still cooling.
4. **Compressor reversal cooldown.** Watch `input_datetime.zc_last_changeover` —
   after a direction change, the unit won't reverse heat↔cool again until
   `min_changeover` minutes pass (unless no room is still calling the current
   direction). With the default of 3 min, the nursery is served ~3 min after the
   last reversal at worst; set the cooldown to 0 for instant reversal.
5. **Relief valve + min-open pressure relief.** Force every room damper closed
   (each switch `on`): `switch.damper_theater` goes **off** (open). Now, **while
   cooling**, open exactly **one** room damper — the theater should **stay off**
   (open) because fewer than 2 zones are open. Open a **second** room and the
   theater goes **on** (closed). (While heating, the theater closes as soon as any
   one room is open — heating is unaffected.)
   - **Guest room mode.** Turn on `input_boolean.theater_guest_mode` and set the
     theater above its cool setpoint: `input_text.theater_demand` goes `cool`, the
     unit runs cool, and `switch.damper_theater` goes **off** (open) to serve it.
     Satisfy the theater (or turn guest mode off) and it reverts to relief-valve
     behavior (open when too few zones are open, per the min-open rule above).
6. **Main on/off.** With a room calling, the unit switches on in the winning
   direction. With every room satisfied/disabled, it switches off.
7. **Sleep / pre-cool.** Turn on `input_boolean.<your_sleep_mode>`; the master
   bedroom uses the 68 °F sleep cool setpoint. The same 68 °F target applies
   automatically inside the 20:00–07:00 window (house occupied) without touching
   the toggle — so an evening reading above 68 begins pre-cooling on its own.
8. **Away.** Turn off `input_boolean.<your_home_occupied>` with nobody in
   `zone.home_wide` (or set the house-mode selector to Away). The eco preset is
   applied and each room switches to its **away** cool setpoint. With the master's
   away setpoint at 75, warm it past 75 → its status shows `Cooling … (away)`, it's
   served (even though unoccupied), and it's held at ~75 instead of drifting to 78.
   Rooms without an away setpoint just use their normal setpoints. Mode = `Eco Away`.
9. **Idle band.** With every room satisfied (nobody calling) and idle behavior
   set to **hold band**, the unit should sit in `heat_cool` on the 64–76 band
   (`Idle - holding 64-76° band` in the status line) — cooling if the house
   exceeds 76, heating below 64, otherwise coasting. (In **cap** mode instead, it
   stays off until the hallway passes 76, then shows
   `Starting cooling (hallway over 76°)`.)
10. **House mode.** Set the house-mode selector to **Away** → within ~2 min the
   unit switches to the eco preset but keeps running. Set it to **Vacation** →
   the unit turns off entirely (server room included). Set it back to **Home** →
   the eco preset is restored and normal operation resumes.
9. **Force-run.** With force-run enabled, set the central thermostat to a mild
   value (e.g. 72) while a room calls for cooling and the hallway is cooler than
   that. The coordinator should drive the target down (≈ sensed − 2) so the unit
   runs, and `input_number.zc_last_cmd_setpoint` updates to that value.
10. **Setpoint override.** Turn `input_boolean.zc_setpoint_manual` on: the
    coordinator stops changing the setpoint (you set it) but still zones. Off →
    it resumes force-run.
11. **Full bypass.** Turn `input_boolean.zc_bypass` on: within the re-sync the
    status lines read `Manual override ON ...`, and now change the thermostat,
    mode, and any damper by hand — nothing gets reverted. Turn it off → automation
    resumes and takes the dampers/unit back over.

### Template sanity check

In **Developer Tools → Template**, paste (substituting your entity IDs) to
confirm the coordinator's core expressions:

```jinja
{% set zone_demands = ['input_text.nursery_demand','input_text.master_demand','input_text.server_demand'] %}
{% set zone_dampers = ['switch.damper_nursery','switch.damper_master_bedroom_1','switch.damper_master_bedroom_2','switch.damper_server_room'] %}
winner_dir:        {% set ns = namespace(dir='none') %}{% for d in zone_demands %}{% if ns.dir=='none' and states(d) in ['heat','cool'] %}{% set ns.dir=states(d) %}{% endif %}{% endfor %}{{ ns.dir }}
all_others_closed: {{ zone_dampers | reject('is_state','off') | list | count == (zone_dampers | count) }}
```

### Reading the status line

If you wired up the optional `input_text.zc_status` helper, the coordinator
describes what it decided in plain English. It **updates only when the decision
changes** (not every cycle), so use the entity's *last changed* time to see when
the coordinator last acted. Format:

```
<what the unit is doing> - <thermostat action> - <house mode>[ - cooling: ...][ - heating: ...]
```

The **thermostat action** part only appears when Setpoint Force-Run is enabled.
The **cooling:/heating:** lists name the rooms currently calling in each
direction (from the demand helpers' friendly names, minus the " demand" suffix);
each is shown only when at least one room is calling that way.

**What the unit is doing:**
- `Starting cooling` / `Starting heating` — unit was off (or idling) and a room called.
- `Cooling` / `Heating` — already running the right direction, left it.
- `Switching to cooling` / `Switching to heating` — reversed for a higher-priority room.
- `Waiting to switch to heating (compressor cooldown 1/3 min)` — a reversal is
  wanted but the compressor cooldown is blocking it (1 of 3 min elapsed).
- `Off - no room calling` — nothing is calling and idle behavior is "off".
- `Idle - holding 64-76° band` — nothing is calling and idle behavior is "hold band".
- `Holding 64-76° band` (with mode `Eco Away`) — nobody home: holding the wide
  band instead of cooling to setpoint, to save power.
- `Off - forced (vacation)` / `Off - forced (away)` — house mode Vacation, or
  away with the turn-off action.
- `... (hallway over 76°)` — cooling only because the hallway passed the cap (no
  room asked), e.g. `Starting cooling (hallway over 76°)`.

**Thermostat action** (force-run):
- `thermostat forced to 67° (hallway is 72°, pushed below it so the unit keeps
  running - this is a lever, not the room target)` — the central thermostat
  senses the **hallway**, so to make the unit run for a room the coordinator
  pushes the setpoint **below** the hallway (for cooling) or **above** it (for
  heating). The number is a *control lever*, **not** the temperature any room is
  aimed at — each room is served by its damper opening/closing. So "67° while the
  hallway reads 72°" just means "keep cooling"; it does **not** mean the house is
  being cooled to 67.
- `manual override - thermostat left alone` — the hard manual override toggle is on.
- `recent manual change - leaving thermostat alone` — only if you enabled the
  (off-by-default) auto manual-edit grace.

**House mode:**
- `Home Day` — home, outside the night window.
- `Home Night` — home, inside the night window (or the night/sleep toggle is on).
- `Eco Away` — nobody home and the away action is the eco preset.
- `Away` — nobody home (away action is "do nothing" or "turn off").
- `Vacation` — house-mode selector set to Vacation.

The Day/Night split comes from the coordinator's **Night starts/ends at** times
(default 21:00 → 07:00) plus the optional **Night/sleep toggle** — both are
label-only and change no conditioning.

Examples:

```
Cooling - thermostat forced to 67° (hallway is 72°, pushed below it so the unit keeps running - this is a lever, not the room target) - Home Day - cooling: Master bedroom
Waiting to switch to heating (compressor cooldown 1/3 min) - Home Day - heating: Nursery - cooling: Master bedroom
Idle - holding 64-76° band - Home Night
Holding 64-76° band - Eco Away
Off - forced (vacation) - Vacation
```

If the status line **never updates**, the coordinator automation itself isn't
running — check that it's enabled and look at its trace. If it shows
`Off - no room calling` while a room is hot, that room isn't publishing demand
(check its demand `input_text` and temp sensor).

### Reading a room's status

Each Room Zone can also write a plain-English line to its own status helper
explaining **why it is or isn't calling** — its temperature vs its *effective*
target, whether it's on day or sleep setpoints, and whether it's yielding or
disabled. Examples:

```
Cooling - 76.4° vs 75.0° target (day)                 ← above the 75° day target, so it's calling
Cooling - 68.5° vs 68.0° target (sleep)               ← on the sleep setpoint (why it cools a ~68° room)
Cooling - 76.0° vs 75.0° target (away)                ← nobody home: held at the away setpoint (not baking)
Idle - 71.0° (cool above 72.0°, heat below 68.0°) (day)  ← comfortable, not calling
Heating - 66.5° vs 68.0° target
Cooling (yielding to priority) - 76.0° vs 75.0° target (day)  ← damper closed to focus air on the nursery
Idle - 73.0° (not eligible: occupancy/schedule)       ← gated off by occupancy or its window
Off (disabled)                                        ← enable toggle is off
No temperature reading                                ← sensor unavailable (fails safe to idle)
```

This is the companion to the coordinator line: the coordinator explains what the
*unit* is doing; each room status explains what that *room* wants and why. The
`(day)` / `(sleep)` / `(away)` tag is the quickest way to catch a stuck sleep
toggle or confirm the away cap is in effect.

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
