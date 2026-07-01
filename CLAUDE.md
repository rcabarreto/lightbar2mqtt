# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`lightbar2mqtt` is Arduino/C++ firmware for an **ESP32 + nRF24L01** that bridges the Xiaomi Mi Computer Monitor Light Bar (Model MJGJD**01**YL only — the 02 with BLE/WiFi won't work) to MQTT and Home Assistant. It transmits the light bar's proprietary 2.4 GHz radio protocol to control it, and sniffs the same protocol to report physical remote actions. Radio protocol details live in the referenced [lamperez/xiaomi-lightbar-nrf24](https://github.com/lamperez/xiaomi-lightbar-nrf24) repo.

## Build & flash

There is no CLI build; this is an Arduino IDE sketch (`Lightbar.ino` is the entry point — the sketch folder name must match).

1. Copy `config-example.h` → `config.h` and edit it. `config.h` is gitignored and required to compile; do not commit it.
2. Install libraries via Arduino Library Manager (exact versions matter):
   - Arduino_JSON `0.2.0`, CRC (Rob Tillaart) `1.0.3`, PubSubClient `2.8`, RF24 (TMRh20) `1.4.10`
3. Select the ESP32 board + serial port, Upload.
4. Serial monitor is **115200 baud**. It prints the device ID, MQTT root topic, and — crucially — the serial of any unrecognized remote it hears (`Ignoring package with unknown serial: 0x...`), which is how you discover your remote's serial to put in `config.h`.

There are no tests and no linter.

## Configuration model

- `config.h` (user-created, gitignored) holds all deployment settings: nRF24 pins, WiFi/MQTT credentials, HA discovery, NTP, and the `LIGHTBARS[]` / `REMOTES[]` arrays of `{serial, name}` pairs (`SerialWithName` in `constants.h`).
- `config_defaults.h` only backfills the radio pin `#define`s if `config.h` omitted them.
- `constants.h` holds compile-time caps (`MAX_LIGHTBARS`, `MAX_REMOTES`, `MAX_SERIALS`, `MAX_COMMAND_LISTENERS`) and the firmware `VERSION`. Bump `VERSION` here on release — it is reported in HA discovery payloads.
- NTP time sync is fully optional and gated by `#if defined(...)` on `GMT_OFFSET_SEC`/`DST_OFFSET_SEC`/`NTP_SERVER` (see `Lightbar.ino`); it only exists to help some routers display the device.

**Serial coupling concept:** if a `LIGHTBARS` entry uses the *same* serial as a `REMOTES` entry, the original physical remote still drives the bar directly ("attached"). Using a *different* serial for the light bar ("detached") lets the ESP32 be the sole controller so remote actions only fire HA events. This distinction shapes much of the state-handling logic.

## Architecture

Ownership: `Lightbar.ino` creates one `Radio`, one `MQTT`, and one `Lightbar`/`Remote` object per config entry, then drives everything from `loop()` (`mqtt.loop()` + `radio.loop()`). Objects are heap-allocated and never freed (device runs forever).

- **`Radio`** — owns the RF24 hardware. Encodes outbound packets (preamble + serial + rolling sequence counter + command + CRC16) and sends each command 20× for reliability (`sendCommand`). On the receive side, `handlePackage()` does the bit-shift decode described in lamperez's docs, validates preamble + CRC, matches the serial against registered `Remote`s, de-duplicates by rolling package id (`PackageIdForSerial`), and calls `remote->callback()`. Radio params (channel 68, 2MBPS, CRC disabled, 17-byte payload, no auto-ack) are fixed in `setup()` and must not be changed casually — they match the light bar's protocol.
- **`Lightbar`** — models one bar. Translates high-level intent to raw radio commands (`Command` enum: ON_OFF, COOLER, WARMER, BRIGHTER, DIMMER, RESET). Brightness/temperature are **relative** on the wire, so `setBrightness`/`setTemperature` first send a max-step in one direction, then N steps back to reach an absolute value. Mireds (153 cold – 370 warm) are mapped to the bar's 0–15 scale in `setMiredTemperature`.
- **`Remote`** — models one physical remote. Registers itself with the `Radio` in its constructor. Holds a list of command-listener callbacks; `Radio` invokes `callback()` on a matching sniffed packet, which fans out to listeners.
- **`MQTT`** — the glue. Wraps PubSubClient; client id / root topic derived from the ESP32 MAC (`l2m_<MAC>`). Subscribes to `<root>/+/{command,pair,toggle_internal}`, parses JSON commands in `onMessage`, drives the matching `Lightbar`, publishes remote actions, and generates all Home Assistant discovery payloads.

### State-management caveat (important)

The protocol is **one-way for state**: the light bar never reports its actual on/off/brightness. `Lightbar` keeps a best-effort `onState` bool that assumes ON at boot. Consequences reflected in the code:

- HA is treated as the source of truth for on/off (see commit history). `publishLightbarState` retains the assumed state; a `toggle_internal` button + topic exists purely to let the user correct an inverted state without power-cycling.
- `setOnOff` only sends ON_OFF if the assumed state differs from the request, so a desynced `onState` can invert everything until corrected.
- Sniffed remote actions are best-effort and occasionally missed (radio limitation) — don't build logic assuming every physical remote action is captured.

## MQTT topic map

Root topic is `<MQTT_ROOT_TOPIC>/l2m_<ESP32 MAC>`. Per-device subtopics key off the lowercase `0x<serial>`:

- `.../0x<lightbar>/command` — inbound JSON: `{"state":"ON|OFF","brightness":0-15,"color_temp":153-370}`
- `.../0x<lightbar>/state` — outbound retained assumed state
- `.../0x<lightbar>/pair` — inbound trigger (power-cycle bar within 10s first)
- `.../0x<lightbar>/toggle_internal` — inbound; flip the assumed on/off state
- `.../0x<remote>/state` — outbound remote action string (`press`, `turn_clockwise`, `hold`, etc.), published then cleared with a NULL to make it edge-like for HA triggers
- `.../availability` — `online`/`offline` (LWT)

Home Assistant discovery: the light bar is published as a `light` entity + `button`s (Pair, Toggle Internal State); the remote as a `sensor` + six `device_automation` triggers. Discovery JSON is built as inline raw-string (`R"json(...)"`) templates in `mqtt.cpp` sharing a `baseConfig` block — edit those carefully, they are hand-assembled strings, not a serializer.

## Conventions

- All modules log to Serial with a `[Module]` prefix (`[Radio]`, `[MQTT]`, `[WiFi]`, `[Time]`, `[Remote]`).
- Capacity overflows (too many remotes/lightbars/serials/listeners) log a message telling the user to raise the relevant `constants.h` cap and recompile, then bail — follow this pattern for new bounded arrays.
- On unrecoverable setup failures (WiFi, MQTT, radio) the code retries ~60× then calls `ESP.restart()` rather than hanging.
