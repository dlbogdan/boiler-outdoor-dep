# Pico 2W Standalone Boiler Weather Compensation — Architecture Draft

## Hardware

```
┌─────────────────────────────────────────────────────────┐
│  Pi Pico 2W  (RP2350, 520KB SRAM, WiFi)                │
│                                                         │
│  GPIO 0,1 (I2C0) ──► OLED 256×64 (SSD1322 or SH1106)  │
│  GPIO 4,5 (I2C1) ──► BH1750 (lux sensor)               │
│  GPIO 6        ──► DS18B20 (outdoor temp, 1-Wire)       │
│  GPIO 7        ──► OpenTherm Adapter (TX)               │
│  GPIO 8        ──► OpenTherm Adapter (RX)               │
│  GPIO 10-14    ──► 5 buttons (UP/DOWN/LEFT/RIGHT/OK)    │
│  WiFi          ──► MQTT broker / HA API (optional)      │
└─────────────────────────────────────────────────────────┘
```

- OpenTherm: OT adapter board (DIYLESS or Ihor Melnyk shield). PIO for Manchester encoding.
- OLED: SSD1322 (SPI, grayscale) or SSD1309 256×64. SPI preferred for refresh speed.
- Lux: BH1750 (65535 lx max, sufficient for 10k–40k thresholds). TSL2591 if higher range needed.
- Temp: DS18B20, 1-Wire on GPIO.

## Software Structure

```
main.py
├── config.py           ← params dict + JSON load/save
├── state.py            ← solar_accum, heating_on + flash persistence
├── sensors/
│   ├── ds18b20.py      ← outdoor temp (+ optional room temp for Option B)
│   └── bh1750.py       ← lux
├── control/
│   ├── heat_curve.py   ← base_flow = f(t_out, params)
│   ├── solar_accum.py  ← ODE step
│   ├── hysteresis.py   ← OFF/HEATING state machine
│   ├── failsafe.py     ← sensor timeout, frost protect
│   └── controller.py   ← orchestrator: 60s tick
├── opentherm/
│   ├── pio_driver.py   ← PIO Manchester TX/RX
│   ├── protocol.py     ← message framing, f8.8 encode/decode
│   └── boiler.py       ← set_ch_setpoint(), read_flow_temp(), status heartbeat
├── ui/
│   ├── display.py      ← OLED framebuf
│   ├── screens.py      ← status / config / failsafe screens
│   └── buttons.py      ← debounce + nav
├── net/
│   └── mqtt.py         ← optional: publish state, receive demand
└── scheduler.py        ← main loop with WDT
```

## Cooperative Scheduler (no RTOS)

```python
while True:
    now = time.ticks_ms()
    sensors.poll(now)          # read temp/lux every 10s
    controller.tick(now)       # recompute every 60s
    ui.tick(now)               # redraw display ~10 fps
    net.tick(now)              # MQTT keepalive + publish (optional)
    ot.heartbeat(now)          # OT status every ~900ms (hard requirement)
    wdt.feed()
    time.sleep_ms(50)          # ~20 Hz main loop
```

## Config

~20 parameters, stored as JSON in flash (littlefs). Editable via button UI.

```python
DEFAULTS = {
    "t_design": -20, "flow_design": 77, "curve_base": 25,
    "b": 0.78, "flow_min": 25, "flow_min_on": 36,
    "t_off": 18.0, "t_on": 13.0, "flow_max": 67,
    "lux_low": 10000, "lux_high": 40000, "lux_max_offset": 5.0,
    "solar_charge": 0.15, "solar_halflife": 25,
    "lux_mult": 1.0, "min_change": 2,
}
```

## Hysteresis State Machine

```
     ┌───────────┐  t_out < t_on AND flow > flow_min_on  ┌──────────┐
     │    OFF    │ ──────────────────────────────────────► │ HEATING  │
     │           │ ◄────────────────────────────────────── │          │
     └───────────┘  t_out ≥ t_off OR flow ≤ flow_min      └──────────┘
                        (dead zone: hold current state)
```

## Demand / P-Controller Options (No HA)

- **Option A (recommended):** Drop P-controller entirely. Heat curve + solar is sufficient.
- **Option B:** Add DS18B20 in main room. Use room temp error as demand signal: `demand_p_offset = clamp(gain * (target_room - actual_room), -max, max)`.
- **Option C:** Zigbee coordinator for TRV positions. Not recommended (immature on MicroPython).

## Failsafes

- **Sensor timeout:** If no outdoor temp reading for 10 min → fixed failsafe flow temp (e.g., 45°C).
- **Frost protection:** If t_out < -5°C → clamp flow temp floor to 35°C. Send OT frost enable (DataID 0).
- **OT heartbeat:** Must send MsgID 0 every ~900ms or boiler reverts to internal control (safe by design).
- **Hardware watchdog:** `WDT(timeout=8000)`, fed in main loop.

## State Persistence

```json
// state.json — written to flash every 5 min or on state change
{
    "solar_accum": 2.3,
    "heating_on": true,
    "last_update_epoch": 1744100000
}
```

On boot: read state, compute elapsed time, decay solar accumulator by the gap, resume.

## Display Layouts (256×64)

### Status screen
```
┌──────────────────────────────────────────┐
│ OUT: -5.2°C  ☀ 23400lx         HEATING  │  16px
│──────────────────────────────────────────│
│ FLOW TARGET:  52°C    (current: 51°C)   │  16px
│ Base: 55  Solar: -2.1  P: -0.9          │  16px
│ Solar Accum: ████████░░ 3.2/5.0°C       │  16px
└──────────────────────────────────────────┘
```

### Config screen
```
┌──────────────────────────────────────────┐
│ ► CONFIG                          3/20   │
│──────────────────────────────────────────│
│   Design Outdoor Temp                    │
│              [ -20 °C ]                  │
│         ◄─────────●─────────►            │
└──────────────────────────────────────────┘
```

## Memory Budget (RP2350, 520KB SRAM)

| Component | Est. RAM |
|---|---|
| Framebuffer (256×64×1bpp) | 2 KB |
| MicroPython heap overhead | ~80 KB |
| Config dict | <1 KB |
| Sensor buffers | <1 KB |
| OpenTherm PIO buffers | <1 KB |
| MQTT + WiFi stack | ~40 KB |
| **Total** | **~125 KB** |

## Notes

- Core control logic is ~200 lines of Python. Bulk of work is OT driver and UI.
- WiFi/MQTT is optional — device works fully standalone with WiFi off.
- Could use regular Pico 2 (no W) if remote monitoring not needed.
- Don't write flash every tick — batch with dirty flag, 5 min interval.
- Use `framebuf.FrameBuffer` + Peter Hinch's `writer.py` for fonts.
