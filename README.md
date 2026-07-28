# PGC_litterbox

Automated self-cleaning pet litter box control system powered by **ESPHome** and **Home Assistant**.

This repository contains the core logic package (`litterbox_core.yaml`) for managing directional motor drivers, infrared cat presence detection, Jam/Overcurrent protection, automatic cycle triggering, and cycle tracking statistics.

---

## Features

* **Automated Scoop Cycle:** Detects when a cat enters and leaves, triggering a delayed cleaning cycle automatically.
* **Concurrency Protection:** Built-in guard checks prevent multiple scoop cycles from running simultaneously (eliminating `Script 'run_scoop_cycle' is already running` log warnings).
* **Smart Safety Interlocks:**
  * **Cat Presence Guard:** Immediately stops the motor if the cat re-enters during a cycle.
  * **Current Limit / Jam Protection:** Monitors motor current draw (via ACS712 or similar) to halt/reverse the scoop if an obstruction is encountered.
* **Manual Controls:** Dedicated template buttons in Home Assistant to trigger manual scoop cycles, park/reset the rake, or clear jam errors.
* **Usage & Health Telematics:** Tracks total daily visits, visit durations, jam counts, and cycle runtimes.

---

## Hardware & Pin Assignments

Default pin mapping used by `litterbox_core.yaml` (overridable via ESPHome `substitutions` in individual node configs):

| Function | Default Substitution Variable | Default GPIO | Notes |
| :--- | :--- | :--- | :--- |
| **Cat IR Sensor** | `${pin_cat_ir}` | `GPIO3` | Active-low / active-high IR presence beam |
| **Motor Forward Relay/PWM** | `${pin_motor_fwd}` | `GPIO4` | Drives scoop forward |
| **Motor Reverse Relay/PWM** | `${pin_motor_rev}` | `GPIO5` | Drives scoop backward |
| **Home Limit Switch** | `${pin_limit_home}` | `GPIO6` | Parks the scoop in resting position |
| **Dump Limit Switch** | `${pin_limit_dump}` | `GPIO7` | Indicates full dump extension |
| **Current Sensor Analog** | `${pin_current_sense}`| `ADC / GPIO1` | Monitors motor current for jam detection |

---

## Installation & Deployment

### 1. Structure

Include `litterbox_core.yaml` directly from GitHub in your main ESPHome node config.

### 2. Sample ESPHome Node Config

```yaml
substitutions:
  device_name: "pricklylitter2"
  friendly_name: "Prickly Litterbox 2"
  # Optional: Override default pin assignments if hardware wiring differs
  # pin_cat_ir: GPIO3

esphome:
  name: ${device_name}
  friendly_name: ${friendly_name}

packages:
  litterbox_core: github://pricklyguy/PGC_litterbox/litterbox_core.yaml@main
  wifi: !include packages/wifi.yaml
  common: !include packages/common_sensors.yaml
  web: !include packages/web.yaml
```
---

### 3 . Entities Exposed to Home Assistant
Upon compilation and flashing, ESPHome exposes the following entities to Home Assistant:

Sensors & Binary Sensors
  binary_sensor.cat_present — Active when cat is inside the box.
  sensor.cat_visit_duration — Renders duration of the most recent visit (seconds).
  sensor.litterbox_motor_current — Real-time motor current draw.
  sensor.litterbox_cycle_status — Reports current state (Idle, Scooping, Returning, Jam Detected).

Buttons & Controls
  button.run_scoop_cycle — Triggers a full clean/dump/return cycle.
  button.stop_motor — Emergency stop for all motor outputs.
  button.clear_jam_error — Resets state machine after a current spike lock.

Safety & Operational Logic
  Cat Entrance: cat_present goes high -> logs entrance timestamp.
  Cat Exit: cat_present clears -> publishes visit duration sensor -> starts delayed scoop cycle script.

Scoop Execution:
  Motor runs forward until limit_dump is hit or current limit threshold is exceeded.
  Pauses briefly over waste bin.
  Motor reverses until limit_home is hit.

Safety Interrupt:
  If cat_present becomes ON at any point during movement, the motor immediately halts and waits until the box is clear before resuming.

License
Distributed under the GPL-3.0 License. See LICENSE for details.
