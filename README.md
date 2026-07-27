# DIY Smart Zone Controller for Ducted Air Conditioning

![Zone Controller](images/zone-controller.png)

Per-room climate zoning for a ducted HVAC system, running **entirely locally** on
Home Assistant — no cloud. Motorized dampers open and close each room's duct
branch based on that room's own temperature, occupancy, and priority, giving one
shared air handler the granularity of a commercial multi-zone system.

It uses an ESP32-based KinCony KC868 relay board to drive 24V AC dampers
(exposed to Home Assistant as `switch` entities), and two Home Assistant
**blueprints** for all of the control logic.

> ⚠️ Work in progress. Expect some rough edges, and test before you deploy.

---

## 📖 Full build & explanation

For the hardware build, the wiring, and the story behind the project:

👉 *[Watch the video here](https://youtu.be/u8PV-tD-EuY)*
👉 *[Read the full write-up here](https://www.thestockpot.net/videos/zone-controller)*

---

## How it all fits together

There are three moving parts. Sensors feed the **Room Zone** blueprint (one copy
per room), which publishes each room's *demand* and drives its damper. The single
**Coordinator** blueprint reads all the demands, decides the whole unit's
direction by priority, and runs the main HVAC unit and the relief valve.

```mermaid
flowchart TD
    subgraph Room["Room Zone blueprint (one per room)"]
        T[Temp sensor] --> RZ
        SP[Setpoints<br/>input_number] --> RZ
        OCC[Occupancy /<br/>Sleep / Enable] --> RZ
        RZ[Room Zone logic]
        RZ -->|writes heat/cool/none| DH[(Demand helper<br/>input_text)]
        RZ -->|open/close| DMP[Room damper switch]
    end

    DH --> CO
    DMP -. state .-> CO

    subgraph Coord["Coordinator blueprint (one)"]
        CO[Priority arbitration<br/>+ anti-short-cycle<br/>+ relief + home/away]
    end

    CO -->|set heat/cool/off| HVAC[Main HVAC<br/>climate entity]
    CO -->|open when all rooms closed| RV[Theater damper<br/>relief valve]
    PRES[Home toggle + wide zone] --> CO
    HVAC -. current mode .-> RZ
```

**Why a demand helper?** A single air handler can only blow hot **or** cold air
at any moment — it can't heat one room while cooling another. So rooms don't just
open their own dampers; they *advertise* what they want, and the Coordinator
picks the winning direction by priority. A room that loses (e.g. it wants cooling
while a higher-priority room is being heated) keeps its damper closed and waits
its turn.

---

## What each module does

### 1. Room Zone blueprint — `blueprints/automation/zone_controller/room_zone.yaml`
Runs once per conditioned room (nursery, master bedroom, server room). For its
room it:
- reads the temperature sensor and compares it to **cool / heat setpoints**
  (with a hysteresis deadband) to compute a raw demand of `heat`, `cool`, or
  `none`;
- respects a **zone mode** — `standard`, `always_on`, or `occupancy`;
- applies an optional **sleep-mode** cool setpoint override;
- honours a hard **enable/disable** toggle;
- writes its demand to an `input_text` helper and opens its damper **only when
  its demand matches the direction the unit is currently delivering**.

> **Damper polarity is inverted / normally-open:** `off` = damper **OPEN**,
> `on` = damper **CLOSED**. A power loss springs every damper open (failsafe).
> The blueprints handle this — you just point them at the right `switch`.

### 2. Coordinator blueprint — `blueprints/automation/zone_controller/zone_coordinator.yaml`
Runs once for the whole house. It:
- reads the room demand helpers **in priority order** and sets the unit's
  direction to the highest-priority room that is calling;
- enforces an **anti-short-cycle** minimum changeover time before flipping
  heat↔cool (unless nothing is still calling in the current direction);
- turns the unit **off** when no room is calling, and back on when one is;
- optionally **force-runs the setpoint** — pushing the central thermostat's
  target past the hallway temperature so the unit runs for a room that needs it,
  while respecting manual setpoint changes (a hard override toggle, or a grace
  period after you edit it by hand);
- drives the **theater damper as a pressure-relief valve** — open only when every
  other damper is closed, so the always-open hallway plus the relief path keep the
  duct from over-pressurizing;
- handles **home/away** (a toggle, or anyone inside a wide ~5 mi zone), defaulting
  to an eco preset so an always-on room keeps running while the house is empty.

### 3. Helper package — `packages/zone_controller.yaml`
Creates the supporting entities: per-room setpoints (`input_number`), enable
toggles and sleep/home toggles (`input_boolean`), per-room demand helpers
(`input_text`), the changeover timestamp (`input_datetime`), and the wide
presence `zone`.

### 4. Firmware — `YAML`
The ESPHome config for the KC868 board that exposes the damper relays to Home
Assistant. Not required reading to use the blueprints, but it's what the `switch`
entities come from.

---

## The rooms

| Priority | Room | Damper(s) | Mode | Setpoints | Enable toggle |
|:--:|------|:--:|------|-----------|:--:|
| 1 | Nursery | 1 | standard, tight band | Heat < 68 °F, Cool > 71 °F | ✅ |
| 2 | Master bedroom | 2 | occupancy, cool-only | Cool > 75 °F day / **> 68 °F sleep** · pre-cools from ~8pm when home | ✅ |
| 3 | Server room | 1 | always-on, cool-only | Cool > 80 °F | ✅ |
| 4 | Theater / bonus | 1 | relief valve | — (automatic) | — |
| — | Hallway | 0 | always open | — | — |

Priority is the **order you list the demand helpers** in the Coordinator. When
directions conflict, the higher-priority room wins the unit: e.g. if the nursery
needs heat while the master needs cooling, the unit heats for the nursery first;
the master waits until the nursery is satisfied (or the changeover timer lets it
switch).

---

## Installation

### Prerequisites
You need these already in Home Assistant:
- **Room temperature sensors** — one per room.
- **Master bedroom occupancy** — a `binary_sensor` that is `on` when occupied.
- **Damper switches** — one `switch` per damper (5 total; master bedroom has 2).
- **Main HVAC** — a `climate` entity supporting `cool`, `heat`, `heat_cool`, `off`.

### 1. Install the helper package
Copy [`packages/zone_controller.yaml`](packages/zone_controller.yaml) to
`<config>/packages/zone_controller.yaml`. Enable packages in
`configuration.yaml` if needed:
```yaml
homeassistant:
  packages: !include_dir_named packages
```
Edit the file to fill in the `# TODO` values (zone latitude/longitude, and check
the setpoint defaults), then **restart Home Assistant**.

### 2. Import the blueprints
**Settings → Automations & Scenes → Blueprints → Import Blueprint**, and import
both:
- `blueprints/automation/zone_controller/room_zone.yaml`
- `blueprints/automation/zone_controller/zone_coordinator.yaml`

(Or copy them into `<config>/blueprints/automation/zone_controller/`.)

### 3. Create one Room Zone automation per room
Create three automations from the **Room Zone** blueprint — nursery, server room,
master bedroom — pointing each at its own temperature sensor, damper switch(es),
setpoints, enable toggle, and demand helper. The theater does **not** get one.
See [`docs/zone-controller-setup.md`](docs/zone-controller-setup.md) for the
exact per-room field values.

### 4. Create the Coordinator automation
Create one automation from the **Coordinator** blueprint. The important field is
**"Room demand helpers, highest priority first"** — add them in priority order:
`input_text.nursery_demand`, `input_text.master_demand`,
`input_text.server_demand`. Also give it the room dampers, the theater (relief)
damper, the changeover helper, and the home/away entities.

### 5. Add dashboard toggles (optional)
Put the enable toggles (`input_boolean.*_enabled`), `input_boolean.zc_sleep_mode`,
and `input_boolean.zc_home_occupied` on a dashboard for easy control.

Full step-by-step field values and a verification checklist are in
**[docs/zone-controller-setup.md](docs/zone-controller-setup.md)**.

---

## Features

- **Per-room zoning** — every room tracks its own setpoints and occupancy.
- **Priority arbitration** — a single air handler is time-shared by room
  priority; the most important room is served first.
- **Enable/disable per room** — flip a toggle to drop a room out of the system.
- **Anti-short-cycle** — a minimum heat↔cool changeover protects the compressor.
- **Setpoint force-run** — pushes the central thermostat's target past the
  hallway temperature so the unit actually runs for a room that needs it, with a
  manual override (hard toggle, or a grace period after a hand edit).
- **Sleep mode** — the master bedroom drops to a colder target at night.
- **Scheduled activation** — a room can pre-cool on a daily window when the
  house is occupied (the master bedroom starts around 8pm for a cold bed).
- **Home/away + proximity** — conditioning is reduced when the house is empty,
  resuming when someone comes within ~5 mi.
- **Pressure relief** — the theater damper opens automatically so the duct is
  never over-pressurized.
- **Fail-open** — normally-open dampers spring open on power loss.
- **100% local** — no cloud dependency.

---

## 💬 Got questions?

Open an issue here or leave a comment on the
[YouTube video](https://www.youtube.com/@TheStockPot-AU). I'll do my best to help.

## License

MIT — use it, break it, fix it, share it.
