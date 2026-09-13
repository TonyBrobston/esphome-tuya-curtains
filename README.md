# esphome-tuya-curtains

ESPHome configs for a fleet of Zemismart ZM-CL02 Tuya-MCU curtain motors (ESP8266/TYWE1S WiFi module).

`common-curtain.yaml` holds the shared base config, pulled into each curtain's top-level `.yaml` file via ESPHome's `packages:`. Each curtain file supplies its own `substitutions:` (name, friendly name, wifi/api/AP secrets, `direction_restore`) and nothing else.

## Secrets

Each curtain's `substitutions:` block names the secrets it needs via `!secret`. Add these to the `secrets.yaml` the ESPHome dashboard already uses (it's gitignored here and lives only in the dashboard's own secret management):

- `wifi_ssid` / `wifi_password` — shared across the fleet.
- `<name>_api_key` — must match whatever `api.encryption.key` the device is already running (the dashboard's onboarding wizard generates one when you adopt the base ESPHome flash). Don't invent a new value here, or the running device and this config will disagree and the API connection will fail. Pull the real value from the device's current config (dashboard -> device -> Edit, or the `/json-config` API).
- `<name>_ap_password` — the fallback AP's password. Unlike the API key, this is new (the AP previously had none), so any secure value works.

`!secret` only resolves against the local `secrets.yaml` for files the dashboard loads directly. Since `common-curtain.yaml` is fetched remotely via `packages:`, its wifi credentials are passed in as substitutions (`${wifi_ssid}` / `${wifi_password}`) rather than referenced directly with `!secret`.

## Hardware / protocol notes

Confirmed live against physical units, not just inferred from documentation:

- **Board**: the chip reports 2MB flash (FlashChipId 1540C8), but `esp01_1m` (1MB board profile) is the well-tested convention for this WB2S-style Tuya curtain module family. Undershooting flash size is safe; it only under-uses the chip.
- **UART**: Tuya-MCU link is 9600 baud, ESP TX = GPIO15, ESP RX = GPIO13. GPIO1/GPIO3 remain free as the normal serial-flashing pins.
- **dp101** (`position_datapoint`): cover position, 0–100, linear, **0 = fully open, 100 = fully closed**. Confirmed by sending raw datapoint writes directly and visually observing the result: raw 0 -> fully open, raw 100 -> fully closed. This matches ESPHome's default (non-inverted) position math, so no `invert_position_report` is needed.
- **dp102** (`control_datapoint`): deliberately left unmapped. This device's dp102 enum is CLOSE=0/OPEN=1/STOP=2, but ESPHome's `tuya` cover hardcodes OPEN=0/STOP=1/CLOSE=2 for `control_datapoint` — using it as-is would send inverted open/close/stop commands. Omitting it drives everything through `position_datapoint` instead, and "stop" works via ESPHome's built-in fallback of re-sending the current position (confirmed live to actually halt the motor mid-travel).
- **dp103** (`switch_datapoint`): confirmed as motor direction control, exposed as the internal "Curtain Direction" switch. `direction_restore` is `ALWAYS_OFF` (the fleet default/"regular" orientation) for every curtain except the one(s) physically mounted/wired in reverse, which need `ALWAYS_ON`.
- **Self-calibration**: these motors self-calibrate their endstops on the first move in each direction after power-on, so a position reading taken before both directions have been freshly exercised in the current power cycle can't be trusted.
- **Entity naming**: the cover's `name: ""` (empty) marks it as the device's "main" entity, so Home Assistant uses the device name alone for the entity_id (e.g. `cover.master_bedroom_curtains`) instead of concatenating the entity's own name onto it.

## Removable WiFi dongle

This curtain model has a removable Tuya WiFi dongle (TYWE1S chip) that can be pulled off the motor and flashed on the bench, fully isolated from motor/mains wiring. Standard serial flash wiring: FT232RL (set to 3.3V) TXD -> chip RX (U0RX), RXD -> TX (U0TX), GND -> GND, VCC -> 3V3; GPIO0 -> GND only while power-cycling to enter the bootloader, removed afterward so it boots normal firmware. Some units expose these as a labeled header, others need wires soldered directly to the TYWE1S chip's own U0TX/U0RX/IO0 pads.
