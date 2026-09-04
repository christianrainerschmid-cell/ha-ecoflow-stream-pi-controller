# Home Assistant EcoFlow STREAM PI Controller

Experimental Home Assistant package for PI-based zero-grid charge/discharge control of EcoFlow STREAM devices.

> **Status:** Beta / experimental  
> **Audience:** Advanced Home Assistant users  
> **Use at your own risk**

This project provides a unified PI controller for EcoFlow STREAM batteries in Home Assistant.

The controller tries to keep the grid power close to a configurable target value by adjusting EcoFlow charge and discharge power.

## Tested setup / reference hardware

This project was developed and tested with the following setup:

- Battery system: EcoFlow STREAM Ultra X / STREAM Ultra class device
- EcoFlow firmware: `v1.0.2.1`
- Home Assistant: automation-based control, tested on Home Assistant `2026.4.x`
- Integration: unofficial EcoFlow BLE Home Assistant integration, `ha-ef-ble`
- ha-ef-ble version used during development: `v0.8.5`
- Grid meter: Home Assistant grid power sensor with this sign convention:
  - positive value = grid import
  - negative value = grid export / feed-in
- PV sensor: Home Assistant PV production power sensor
- Optional EV charger sensor: Home Assistant EV charger power sensor

The package is intentionally written with an adapter layer, so other users can map their own entity IDs to the internal controller aliases.

## Important acknowledgement

This project depends on the work done in the EcoFlow BLE integration for Home Assistant:

- rabits / ha-ef-ble  
  https://github.com/rabits/ha-ef-ble

Without this integration and the work behind it, this controller would not be possible in this form.

The `ha-ef-ble` integration provides the practical Home Assistant access to EcoFlow STREAM entities that this controller builds upon.

The BLE integration is the key enabler here. It provides local access to control entities such as charging power limit, base-load/discharge power, energy strategy and backup reserve. Without that local BLE access, a responsive PI-based control loop would not be realistic enough for this use case.

## Why this project exists

The EcoFlow STREAM system is very capable hardware, but for advanced Home Assistant control there is currently no simple, official, fully bidirectional local control interface that behaves like a classic battery inverter API.

In practice, charge and discharge control are exposed through separate entities and modes:

- charging power limit
- base-load / discharge power
- energy strategy / operating mode
- backup reserve

Because of this, Home Assistant users cannot simply send one clean signed setpoint like:

```text
+500 W = discharge
-500 W = charge
```

This project adds that missing abstraction layer in Home Assistant.

It converts the separated EcoFlow controls into one unified control model:

```text
unified_power > 0  => discharge battery
unified_power < 0  => charge battery
unified_power = 0  => idle / deadband
```

On top of that unified model, the package implements a PI controller for near-zero-grid behavior.

The goal is not to replace EcoFlow firmware logic. The goal is to make the STREAM device usable as a controllable energy component inside Home Assistant, while respecting the practical limitations of the available EcoFlow control entities.

## Concept

The controller uses one unified target power value:

```text
unified_power > 0  => discharge battery
unified_power < 0  => charge battery
unified_power = 0  => idle / deadband
```

Grid power convention:

```text
grid_power > 0  => grid import
grid_power < 0  => grid export / feed-in
```

The main control formula is:

```text
error = grid_power - target_grid
p_term = kp * error
i_term = accumulated and limited error correction

target_power = current_unified_power + p_term + i_term
```

This is an **incremental PI controller**. It does not directly calculate an absolute output from scratch. Instead, it adjusts the current EcoFlow output step by step.

## Features

- Unified charge/discharge target power
- PI control with configurable `Kp` and `Ki`
- Configurable target grid value
- Deadband to avoid unnecessary actuator changes
- Anti-windup handling for charge/discharge limits
- Separate charge and discharge step sizes
- Deterministic switching between charge and discharge
- PV-based charge gate
- SOC protection
- Optional EV support mode
- Home Assistant package structure
- Adapter layer for user-specific entity mapping
- Optional energy accounting for estimated PV/grid battery charging share

## What this controller does

The controller can:

- discharge the EcoFlow battery when the house imports from the grid
- charge the EcoFlow battery when there is stable PV surplus
- avoid fast charge/discharge ping-pong
- stop discharge when the battery SOC is too low
- optionally discharge at a fixed power when an EV is charging
- keep a small configurable grid bias, for example `-20 W` or `+20 W`

## What this controller does not do

This is not an official EcoFlow integration.

It does not provide true native bidirectional inverter control. The controller emulates a unified bidirectional setpoint by coordinating the available EcoFlow STREAM entities and switching logic in Home Assistant.

It does not guarantee perfect zero import or zero export. Real systems have latency, sensor noise, BLE/cloud update delays, inverter response delays, and actuator limitations.

This controller is intended as a practical Home Assistant automation approach, not as a certified grid control system.

## Requirements

You need a working Home Assistant setup with:

- EcoFlow STREAM entities available in Home Assistant
- writable EcoFlow charge power entity
- writable EcoFlow discharge/base-load power entity
- writable EcoFlow backup reserve entity
- writable EcoFlow energy strategy select entity
- grid power sensor
- PV power sensor
- battery SOC sensor
- optional EV charger power sensor

The package expects these logical inputs:

```text
grid power sensor
PV power sensor
battery SOC sensor
EV charger power sensor
EcoFlow charging power limit number entity
EcoFlow base-load / discharge number entity
EcoFlow backup reserve number entity
EcoFlow energy strategy select entity
```

## Installation

### 1. Enable Home Assistant packages

In your `configuration.yaml`, make sure packages are enabled:

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Restart Home Assistant after changing this.

### 2. Copy the package file

Copy the package file to:

```text
/config/packages/ecoflow_stream_pi_controller.yaml
```

### 3. Configure the adapter layer

At the beginning of the YAML file, replace the placeholder entities with your own entities.

Example placeholders:

```yaml
sensor.your_grid_power_sensor
sensor.your_pv_power_sensor
sensor.your_battery_soc_sensor
sensor.your_ev_charger_power_sensor

number.your_ecoflow_charging_power_limit
number.your_ecoflow_base_load_power
number.your_ecoflow_backup_reserve
select.your_ecoflow_energy_strategy
```

The rest of the controller uses internal alias entities and scripts. In normal use, you should only need to edit the adapter layer.

### 4. Restart Home Assistant

After adding or changing the package, restart Home Assistant.

Then check that the internal EFCTRL alias sensors are valid and show realistic values.

Important alias sensors:

```text
sensor.efctrl_grid_power
sensor.efctrl_pv_power
sensor.efctrl_battery_soc
sensor.efctrl_ev_power
sensor.efctrl_current_charge_limit
sensor.efctrl_current_discharge_limit
sensor.efctrl_backup_reserve
sensor.efctrl_energy_strategy
```

Do not enable the controller until these values are correct.

## Recommended beta start values

These values are intentionally conservative:

```text
Kp               = 0.20
Ki               = 0.002
Target Grid      = -20 W
Deadband         = 50 W
Integral Limit   = 120 W
Control interval = 5 s
Max Discharge    = 800 W
Max Charge       = 1200 W
Discharge Step   = 10 W
Charge Step      = 1 W
```

Reasonable first goal:

```text
stable behavior around the target range
no aggressive oscillation
no charge/discharge ping-pong
```

Do not start with aggressive PI values.

## Helper entities

The package creates several Home Assistant helpers.

Important helpers:

```text
input_boolean.ecoflow_unified_controller_enabled
input_boolean.ecoflow_unified_charge_allowed
input_boolean.ecoflow_ev_discharge_800w

input_number.ecoflow_unified_kp
input_number.ecoflow_unified_ki
input_number.ecoflow_unified_integral
input_number.ecoflow_unified_integral_limit
input_number.ecoflow_unified_target_grid
input_number.ecoflow_unified_deadband
input_number.ecoflow_unified_max_discharge
input_number.ecoflow_unified_max_charge
input_number.ecoflow_unified_min_discharge_soc
input_number.ecoflow_unified_charge_enable_pv_w
input_number.ecoflow_unified_charge_disable_pv_w
input_number.ecoflow_unified_charge_enable_export_w
input_number.ecoflow_unified_discharge_step_w
input_number.ecoflow_unified_charge_step_w
input_number.ecoflow_unified_ev_threshold_w
input_number.ecoflow_unified_ev_discharge_power

input_button.ecoflow_unified_integral_reset
```

## Safety behavior

When the controller is disabled:

```text
charging power = 0 W
discharge/base-load power = 0 W
integral value = 0 W
```

When the charge gate is blocked:

```text
charging is stopped
discharge is not forced to zero
integral value is reset
```

When SOC protection is active:

```text
discharge is stopped
charging is stopped if needed
integral value is reset
```

When EV charging is detected and EV support is disabled:

```text
EcoFlow output is forced to 0 W
integral value is reset
```

When EV support is enabled:

```text
EcoFlow switches to self-powered mode
backup reserve is set to 20 percent
battery discharges at the configured EV support power
```

Default EV support power:

```text
800 W
```

## Charge gate

Charging is only allowed when PV surplus appears stable.

Enable condition:

```text
PV power > Charge Enable PV
and
grid power < -Charge Enable Export
for 60 seconds
```

Disable condition:

```text
PV power < Charge Disable PV
for 60 seconds
```

Default values:

```text
Charge Enable PV      = 250 W
Charge Disable PV     = 150 W
Charge Enable Export  = 80 W
```

This avoids charging the battery from the grid when PV production is too low.

## PI controller details

### P term

The proportional term reacts to the current grid error.

```text
P = Kp * error
```

A higher `Kp` reacts faster, but can cause oscillation.

### I term

The integral term corrects persistent residual error over time.

```text
I = I + Ki * error * dt
```

The I term is limited by `input_number.ecoflow_unified_integral_limit`.

The I term is reduced or reset in several cases:

- EV charging mode
- charge gate blocked while export would request charging
- actuator saturation
- sign change of the control error
- error inside the deadband
- SOC protection

### Anti-windup

Anti-windup prevents the integral term from continuing to grow when the EcoFlow is already at its configured charge or discharge limit.

Example:

```text
EcoFlow already discharging at max power
and
the error still requests more discharge
=> integral value is frozen
```

Same logic applies to charging.

## Actuator mapping

The package uses scripts as actuator adapters:

```text
script.efctrl_set_charge_power
script.efctrl_set_discharge_power
script.efctrl_set_backup_reserve
script.efctrl_set_energy_strategy
```

This keeps the actual EcoFlow entity IDs in one place.

The controller automation calls only these scripts.

## Actuator write behaviour

Every value sent to the EcoFlow device goes through one of the four `efctrl_set_*` scripts. The controller already avoids unnecessary writes:

- charge limit and base load are written only when `|target - current| >= step`
- inside the deadband the setpoint does not change, so nothing is written
- energy strategy and backup reserve are written only on an actual state change

### Write counters

These counters increment only when a value is actually sent to the device:

```text
counter.efctrl_writes_charge
counter.efctrl_writes_discharge
counter.efctrl_writes_backup_reserve
counter.efctrl_writes_strategy
sensor.efctrl_actuator_writes_total
```

They are cumulative and are never reset automatically. To measure your own write rate, note the total, wait a few days and divide by the number of days.

### Write rate limit

```text
input_number.ecoflow_unified_min_write_interval
```

Minimum number of seconds between two writes to the same actuator entity.

```text
0        = disabled (default, original behaviour)
10 - 20  = reasonable setting to reduce device parameter writes
```

The rate limit is deliberately bypassed for:

- stop commands (`value: 0`)
- calls with `force: true` (mode transitions, e.g. EV support)
- energy strategy and backup reserve, which are state-change writes only

A suppressed write is retried on the next control cycle. Because the PI controller is incremental and re-reads the current EcoFlow output every cycle, no correction is lost — it is only deferred.

The last write timestamp is stored in `input_datetime.efctrl_last_write_*`. A script's own `last_triggered` attribute cannot be used for this, because Home Assistant sets it when the script starts, which would make the script block its own call.

### Note on non-volatile memory

EcoFlow's firmware is closed-source, so it is not known whether BLE parameter updates are held in RAM or committed to flash on every write. The parameters do survive a power cycle, so they reach non-volatile storage at some point.

The worst case can be ruled out empirically: at a 5 s loop, unbuffered writes would exhaust a 10k-cycle flash within days, and STREAM devices have been controlled at these intervals for well over a year without reports of failures. The counters above let you replace this reasoning with an actual measurement for your own system.

If you want to reduce writes further, the largest single factor is `Charge Step`. The default of 1 W makes the charge branch write on nearly every tick when PV output fluctuates; 10-25 W reduces this substantially at very little cost in control quality.

## Testing checklist

Before enabling the controller:

- Check YAML configuration
- Restart Home Assistant
- Verify all EFCTRL alias sensors
- Verify charge power script manually with a safe value
- Verify discharge power script manually with a safe value
- Verify backup reserve script manually
- Verify energy strategy script manually
- Set conservative PI values
- Reset the integral value
- Start with a high deadband if needed

Suggested first test:

```text
Controller enabled
Kp = 0.20
Ki = 0.002
Target Grid = -20 W
Deadband = 50 W
Integral Limit = 120 W
EV support disabled
```

Watch:

```text
grid raw
grid smoothed
error
P term
I term
target power
current EcoFlow power
charge/discharge actuator values
SOC
charge gate
```

## Troubleshooting

### Controller does nothing

Check:

- `input_boolean.ecoflow_unified_controller_enabled` is on
- EFCTRL alias sensors are valid
- target mode is not `disabled`
- SOC is above minimum discharge SOC
- charge gate is allowed when charging is expected
- EcoFlow entities are writable

### Battery does not charge

Check:

- charge gate is `allowed`
- PV power is above enable threshold
- grid power shows export as negative value
- EcoFlow strategy can be switched to `scheduled`
- charging power limit entity accepts values

### Battery does not discharge

Check:

- SOC is above minimum discharge SOC
- EcoFlow strategy can be switched to `self_powered`
- backup reserve can be set
- base-load / discharge entity accepts values

### System oscillates

Try:

```text
lower Kp
lower Ki
increase deadband
increase grid smoothing window
lower integral limit
use a small grid bias instead of exact 0 W
```

Example safer values:

```text
Kp = 0.15
Ki = 0.001
Deadband = 70 W
Target Grid = -20 W
Integral Limit = 80 W
```

## Dashboard

An optional dashboard YAML can be provided separately.

Recommended dashboard sections:

- live status
- PI values
- charge gate
- EV status
- actuator states
- safety states
- history graphs
- logbook

The controller itself does not require the dashboard.

## File structure

Recommended repository structure:

```text
ha-ecoflow-stream-pi-controller/
├─ README.md
├─ LICENSE
├─ packages/
│  └─ ecoflow_stream_pi_controller.yaml
├─ dashboards/
│  └─ ecoflow_stream_pi_dashboard.yaml
└─ docs/
   └─ controller_logic.md
```

## Inspiration / references

This controller is built on top of and inspired by several important sources:

1. EcoFlow BLE integration for Home Assistant  
   rabits / ha-ef-ble  
   https://github.com/rabits/ha-ef-ble

   Without this integration and the work behind it, this controller would not be possible in this form. It provides the practical Home Assistant access to EcoFlow STREAM entities that this package builds upon.

2. General PI/PID controller theory  
   Proportional-integral-derivative controller  
   https://en.wikipedia.org/wiki/Proportional%E2%80%93integral%E2%80%93derivative_controller

3. The idea of an incremental PI-based zero feed-in controller  
   SunEnergyXT / Zero-Feed-in-Controller.yaml  
   https://gist.github.com/SunEnergyXT/e5487ff1669ed17c7a14e00ca2390f75

This EcoFlow STREAM implementation is not a direct copy.

It adapts the incremental PI idea to EcoFlow STREAM devices in Home Assistant, using a unified target power model and additional logic for EcoFlow-specific mode switching, PV charge gating, SOC protection, EV support and anti-windup handling.

## Disclaimer

This is experimental automation code.

Use it only if you understand what it does.

Incorrect configuration may cause unwanted charging, discharging, grid import, grid export, or battery behavior.

You are responsible for testing, validating, and operating this automation safely in your own Home Assistant environment.

This project is not affiliated with EcoFlow, Home Assistant, or the authors of the referenced projects.

## License

MIT License is recommended for this project.

