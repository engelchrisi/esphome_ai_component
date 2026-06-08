---
name: project-overview
description: What esphome_ai_component is, its components, and how they fit together
metadata:
  type: project
---

# esphome_ai_component

GitHub: https://github.com/nliaudat/esphome_ai_component  
Local clone: d:\GitHub\esphome_ai_component

## Purpose
ESPHome external components for reading utility meters (water, gas, electricity) using
an ESP32-S3 + OV2640/OV5640 camera. Two modes:
- **Option A** — `meter_reader_tflite`: AI inference via TFLite Micro (requires model file)
- **Option B** — `ssocr_reader`: seven-segment OCR, no AI model required

## Key Components

| Component | Role |
|---|---|
| `ssocr_reader` | Seven-segment OCR; extends PollingComponent; has `debug: true` flag |
| `meter_reader_tflite` | TFLite inference pipeline; also PollingComponent |
| `esp32_camera_utils` | JPEG decode, crop/scale/rotate; provides `camera_window` hardware zoom |
| `flash_light_controller` | Pre/post flash timing; accepts any `light::LightState*` |
| `value_validator` | Rejects impossible reading jumps (rollover artefacts, OCR noise) |
| `analog_reader` | Analog meter reading (dial-type) |
| `data_collector` | Stores low-confidence readings for model retraining |
| `tflite_micro_helper` | TFLite Micro runtime helper |

## Component Wiring (ssocr_reader)
```
OV5640/OV2640 → esp32_camera (ESPHome built-in)
             → esp32_camera_utils (crop/rotate/scale)
             → ssocr_reader (OCR) → value_validator → HA sensor
flash_light_controller (timing) → ssocr_reader capture callback
```

## Config Entry Points
- `config.yaml` — main reference config for meter_reader_tflite
- `esp32_camera.yaml` — shared camera hardware config (auto-detects OV2640/OV5640)
- `camera_options.yaml` — runtime camera tuning via HA sliders
- `components/*/README.md` — per-component documentation
- `boards/` and `common/boards/` — board pin definitions

**Why:** Central reference for understanding the project without re-reading source each session.
**How to apply:** Use when answering questions about component capabilities, config options, or architecture.
