# Boiler Weather Compensation with Solar Gain

A Home Assistant **blueprint** that automatically adjusts boiler flow temperature based on **outdoor temperature** and **sun illumination (lux)**. No room temperature setpoint is used — each zone manages its own temperature via TRVs/thermostats. The boiler simply produces water at the right temperature for the outdoor conditions.

All sensors and the boiler entity are selected via the UI with proper entity pickers — no YAML editing required.

## How it works

### Heat Curve (Anchor-Point Model)

The flow temperature is interpolated between two user-defined anchor points:

| Anchor | Outdoor Temp | Flow Temp |
|--------|-------------|----------|
| Design day (coldest) | e.g. −5 °C | e.g. 50 °C |
| Warm day (heating off) | e.g. 18 °C | min flow (e.g. 25 °C) |

The interpolation uses a power-law exponent of **0.78** to compensate for the nonlinear heat output of radiators (convection scales as $\Delta T^{1.3}$, so the inverse mapping is $\approx 0.78$):

$$demand = \frac{T_{off} - T_{out}}{T_{off} - T_{design}}$$

$$T_{flow} = T_{flow,min} + (T_{flow,design} - T_{flow,min}) \times demand^{0.78}$$

This is the same physics behind the Vaillant/Kühne heat curves, but expressed as a direct outdoor-to-flow mapping without any room temperature in the equation.

### Solar Gain Offset

On sunny days, solar radiation through windows provides free heating. The automation **reduces** the flow temperature proportionally to outdoor illuminance:

| Lux | Effect |
|-----|--------|
| < 10,000 lx | No reduction |
| 10,000–40,000 lx | Linear ramp from 0 to max offset |
| > 40,000 lx | Full reduction (default: −5 °C) |

This prevents the common problem of overheating on cold but sunny days.

## Sensors Required

You need two outdoor sensors and a controllable boiler in Home Assistant. You select them from dropdown pickers when creating the automation from the blueprint:

| Input | What to select |
|-------|---------------|
| Outdoor Temperature Sensor | Your outdoor thermometer (device class: `temperature`) |
| Outdoor Illuminance Sensor | Your outdoor lux sensor (device class: `illuminance`) |
| Boiler Climate Entity | Your boiler / heat pump `climate` entity |

## Installation

### Option A: Import from GitHub (recommended)

1. Go to **Settings → Automations & Scenes → Blueprints**
2. Click **Import Blueprint**
3. Paste the URL:
   ```
   https://github.com/dlbogdan/boiler-outdoor-dep/blob/main/boiler_weather_compensation.yaml
   ```
4. Click **Preview** then **Import**
5. Go to **Automations** → **Create Automation** → choose the blueprint
6. Select your sensors and boiler, adjust parameters, and save

### Option B: Manual file copy

1. Download `boiler_weather_compensation.yaml` from this repo
2. Place it in `config/blueprints/automation/boiler_weather_compensation/` on your HA instance
3. Restart Home Assistant
4. Go to **Automations** → **Create Automation** → choose the blueprint
5. Select your sensors and boiler, adjust parameters, and save

### Legacy: Package file (optional)

The standalone `boiler_weather_compensation.yaml` package file is still available if you prefer the older helper-based approach. See the file header for installation instructions.

## Configurable Parameters

All parameters are set when creating the automation from the blueprint (and can be changed anytime by editing the automation):

| Parameter | Default | Description |
|-----------|---------|-------------|
| Design Outdoor Temp | −20 °C | Coldest expected outdoor temperature (design day) |
| Design Flow Temp | 77 °C | Flow temperature needed at the design outdoor temp |
| Min Flow Temp | 25 °C | Floor for flow temperature (used at heating-off threshold) |
| Outdoor Heating Off | 18 °C | Above this outdoor temp, heating turns off |
| Lux Low Threshold | 10,000 lx | Illuminance where solar offset begins |
| Lux High Threshold | 40,000 lx | Illuminance where solar offset is at maximum |
| Max Solar Offset | 5 °C | Maximum flow temp reduction due to sun |

## Dashboard Example

The blueprint automation logs all decisions to `system_log`. To see current values at a glance, you can create template sensors in your `configuration.yaml` that read the same outdoor sensors, or simply watch the automation trace in **Settings → Automations → (your automation) → Traces**.

## How to Tune

1. **Set Design Outdoor Temp** to the coldest temperature you expect (e.g. −10 °C typical, −20 °C for extremes).
2. **Set Design Flow Temp** to the flow temperature your radiators need on that coldest day to keep the house warm. Calibrated default: 77 °C at −20 °C (gives ~54 °C at 0 °C outdoor).
3. If the house is **too cold** on cold days → increase Design Flow Temp.
4. If the house is **too warm** on cold days → decrease Design Flow Temp.
5. If the house **overheats on sunny days** → decrease lux thresholds or increase max solar offset.
6. Monitor the automation traces to verify sensible flow temperature values before relying on it.

## Files

| File | Purpose |
|------|---------|
| `boiler_weather_compensation.yaml` | **Blueprint** — import this into Home Assistant |
| `legacy/package_version.yaml` | Legacy package (standalone, uses helpers) |

## References

- [Vaillant Heat Pump Controls: Part 1 – The Heat Curves](https://protonsforbreakfast.wordpress.com/2024/10/16/vaillant-heat-pump-controls-part-1-the-heat-curves/)
- [Part 3: Formulas and Spreadsheet](https://protonsforbreakfast.wordpress.com/2024/10/18/vaillant-heat-pump-controls-part-3-formulas-and-spreadsheet/)
- [André Kühne's formula derivation](https://community.openenergymonitor.org/t/vaillant-arotherm-owners-thread/21891/281)
