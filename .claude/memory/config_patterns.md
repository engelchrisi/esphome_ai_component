---
name: config-patterns
description: Key patterns, coordinate systems, timing values, and gotchas for this project
metadata:
  type: project
---

# Config Patterns & Gotchas

## Two Separate Coordinate Systems
`camera_window` and `crop_x/y/w/h` live in completely different spaces:

| Setting | Space | Units |
|---|---|---|
| `esp32_camera_utils: camera_window` | Raw sensor pixels | OV5640: 2592×1944; OV2640: 1600×1200 |
| `ssocr_reader: crop_x/y/w/h` | Delivered frame pixels | Always 640×480 (camera_resolution) |

`camera_window` zooms into the sensor; the result is scaled to fill `camera_resolution`.
`crop_x/y/w/h` then cuts a rectangle out of that 640×480 frame for OCR.

## OV5640 vs OV2640 flash_pre_time
At `camera_max_framerate: 1fps` the AEC updates once per second:
- **OV2640:** 2000–3000 ms (converges in ~2 frames)
- **OV5640:** 3000–4000 ms (converges in ~3–4 frames); 7000 ms is safe but conservative

`aec2: true` in `esp32_camera.yaml` is OV5640-only; silently ignored on OV2640.

## OTA Pause Pattern (solves slow OTA uploads)
Heavy OCR (JPEG decode + binarisation + connected components) competes with OTA for
PSRAM bandwidth and CPU, causing OTA to stall around 30% (from ~6%/s to ~0.2%/s).

Fix: pause `ssocr_reader` (or `meter_reader_tflite`) during OTA via `on_begin`/`on_end`/`on_error`
callbacks on the `ota: - platform: esphome` block. Result: OTA dropped from ~11+ min to ~11 s.

```yaml
ota:
  - platform: esphome
    password: ${ota_password}
    on_begin:
      then:
        - lambda: |-
            id(${id_prefix}_ssocr_main).set_update_interval(4294967295UL);
            ESP_LOGI("ocr", "OTA started — OCR paused");
    on_end:
      then:
        - lambda: |-
            id(${id_prefix}_ssocr_main).set_update_interval(id(${id_prefix}_ocr_interval_ms));
            ESP_LOGI("ocr", "OTA complete — OCR resumed at %u ms", id(${id_prefix}_ocr_interval_ms));
    on_error:
      then:
        - lambda: |-
            id(${id_prefix}_ssocr_main).set_update_interval(id(${id_prefix}_ocr_interval_ms));
            ESP_LOGI("ocr", "OTA failed — OCR resumed at %u ms", id(${id_prefix}_ocr_interval_ms));
```

Note: `device_base.yaml` also defines `ota: - platform: esphome`. ESPHome merges same-platform
entries from packages, so both coexist. If a "duplicate component" compile error appears,
remove the OTA entry from `device_base.yaml`.

## Global for OCR Interval (single source of truth)
`update_interval:` in ssocr_reader is compile-time only — cannot reference a runtime global.
Workaround: define a global, apply it at `on_boot` (priority -200), use it in all lambdas.
The YAML `update_interval:` value becomes a compile-time placeholder only.

```yaml
globals:
  - id: ${id_prefix}_ocr_interval_ms
    type: uint32_t
    initial_value: '60000'   # only place to change the interval

esphome:
  on_boot:
    priority: -200.0
    then:
      - lambda: id(${id_prefix}_ssocr_main).set_update_interval(id(${id_prefix}_ocr_interval_ms));
```

## OCR Pause Switch
```yaml
switch:
  - platform: template
    name: "OCR Processing"
    optimistic: true
    restore_mode: RESTORE_DEFAULT_ON
    turn_on_action:
      - lambda: id(${id_prefix}_ssocr_main).set_update_interval(id(${id_prefix}_ocr_interval_ms));
    turn_off_action:
      - lambda: id(${id_prefix}_ssocr_main).set_update_interval(4294967295UL);
```

## debug: true vs logger level
- `debug: true` on `ssocr_reader` / `flash_light_controller` / `esp32_camera_utils` enables
  component-internal verbose logging guarded by `if (this->debug_)`.
- A `logger: logs: ssocr_reader: DEBUG` override is redundant when global `log_level` is already DEBUG.

## flash_light_controller accepts any light::LightState*
No platform restrictions. Switching from single LED (`num_leds: 1`) to a ring (`num_leds: N`)
only requires changing `num_leds` — no component code changes needed.

**Why:** These patterns come up repeatedly when configuring or debugging the component.
**How to apply:** Reference before suggesting config changes to avoid re-discovering these gotchas.
