---
name: user-preferences
description: User's preferred working style and feedback from this session
metadata:
  type: feedback
---

# User Preferences

## File Access
- Can read and write `\\192.168.20.204\config\esphome\esp-watermeter-house.yaml` directly.
- Shared package files (`common/device_base.yaml`, board YAMLs, `esphome_ai_component/*.yaml`)
  are used by multiple devices — edit only when necessary and note the impact.

**Why:** User confirmed direct file access is allowed; shared files need care.
**How to apply:** Edit the device YAML freely; flag changes to shared files before making them.

## Comment Style in YAML
User likes wiring documentation kept as comments directly in the YAML light/component block.
When hardware options are switched out (e.g. ring → internal LED), preserve the alternative
as a commented block with step-by-step re-enable instructions rather than deleting it.

**Why:** User asked to keep CJMCU ring wiring as comments when switching back to internal LED.
**How to apply:** Don't delete hardware config — comment it out with instructions.

## Single Source of Truth
User wants runtime-configurable values defined once (global variable) rather than hardcoded
in multiple lambdas. Applies to OCR update interval and likely other repeated values.

**Why:** User explicitly asked "can you replace the 60s also with the global."
**How to apply:** When a value appears in more than one lambda, propose a global.

## OV5640 / OV2640 Switching
User switches between camera sensors. Keep OV5640-specific settings as labelled comments
so they can be re-enabled without re-deriving values. Key differences to flag on switch:
- `flash_pre_time` (2000 ms OV2640 vs 3000–4000 ms OV5640)
- `camera_window` coordinates (sensor pixel space: OV2640 1600×1200 vs OV5640 2592×1944)
- `aec2: true` (OV5640 only, ignored on OV2640)

**Why:** User switched sensors mid-session and asked to preserve OV5640 config as comments.
**How to apply:** When camera type changes, update `flash_pre_time` and label commented blocks.
