---
name: user-hardware-config
description: User's ESP32-S3 water meter device — hardware, pins, and YAML config location
metadata:
  type: user
---

# User Hardware & Config — ESP Watermeter House

## Config File
`\\192.168.20.204\config\esphome\esp-watermeter-house.yaml`  
(Home Assistant config share — readable and writable by Claude)

## Shared Package Files (shared across devices — edit carefully)
- `\\192.168.20.204\config\esphome\common\device_base.yaml` — logger, api, ota, web_server, time
- `\\192.168.20.204\config\esphome\common\boards\esp32s3_camera_n16r8.yaml` — pin substitutions
- `\\192.168.20.204\config\esphome\esphome_ai_component\esp32_camera.yaml` — camera hardware config

## Hardware
- **Board:** Generic ESP32-S3-WROOM-1 N16R8 (16 MB flash, 8 MB octal PSRAM)
- **Camera (current):** OV2640
- **Camera (alternate):** OV5640 — has commented-out settings for easy switch-back
- **Flash LED (current):** Board internal WS2812 on GPIO48, `num_leds: 1`
- **Flash LED (alternate):** CJMCU-2812-7 ring, 7× WS2812B, GPIO38 — documented as comments in YAML

## Key Pin Assignments (from board YAML)
| Pin | Use |
|---|---|
| GPIO2 | Status LED |
| GPIO4 | Camera I2C SDA |
| GPIO5 | Camera I2C SCL |
| GPIO6 | VSYNC |
| GPIO7 | HREF |
| GPIO8–12, 16–18 | Camera data D0–D7 |
| GPIO13 | PCLK |
| GPIO15 | XCLK |
| GPIO48 | Onboard WS2812 flash LED |
| GPIO38 | Free — used for CJMCU ring when active |

## OCR Setup
- **Component:** `ssocr_reader` (Option B, no AI)
- **Meter:** Water meter, m³, 8 digits (5 integer + 3 decimal)
- **Resolution:** 640×480 JPEG, 1 fps max, 0.2 fps idle
- **Update interval:** 60 s — controlled via global `watermeter_h_ocr_interval_ms = 60000`
- **Flash pre-time:** 2000 ms (OV2640); use 3000–4000 ms for OV5640
- **Crop:** Still full-frame (0,0,640,480) — needs tuning after camera mounting

## Runtime Controls in HA
- **"OCR Processing" switch** — pauses/resumes OCR via `set_update_interval`
- Camera flip/mirror/exposure/WB sliders via `camera_options.yaml`

**Why:** Avoids re-reading the config file to recall basic hardware facts.
**How to apply:** Reference when suggesting config changes, pin assignments, or camera settings.
