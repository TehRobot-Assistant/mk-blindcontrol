# About MK-BlindControl

An ESP8266-powered venetian-blind tilt controller. One WiFi board, one or two MG90S servos, MQTT + Home Assistant / openHAB out of the box, and a web UI on the device for setup, alignment, and limit-setting.

This page is the "Wiki / About" link from the firmware's footer. It exists to give credit where it's due, point you at the authoritative build notes upstream, and summarise what changed in this fork.

## Who built what

Three people's work is baked into every board that runs this firmware. Each built on what came before.

### Matt Kaczynski — MK Smarthouse (original design)

Matt designed the **original hardware + firmware** and wrote the definitive build guide:

> **MK Smarthouse — Blinds Control (MQTT / openHAB / ESP8266)**  
> <https://www.mksmarthouse.com/blogs/control/blinds-control-mqtt-openhab-esp8266>

If you're wiring a new unit, start there. It covers the PCB layout, the servo + magnet mechanical hack, the 3D-printed mount, the MQTT topic structure, and the reasoning behind every choice. The servo-arm + rubber-band return mechanism in particular is a genuinely clever piece of engineering — it lets a cheap MG90S hobby servo drive a venetian-blind tilt mechanism without geartrain slop, and it's the reason this design still works ten years after the first board shipped. Matt's shop at <https://www.mksmarthouse.com/> still sells the finished product if you don't want to build one yourself.

### mountain-pitt — V7 / V8 (upstream)

`mountain-pitt` took Matt's design and **kept it alive as open source**, shipping the V7 and V8 firmware on GitHub:

> **mountain-pitt / mk-blindcontrol**  
> <https://github.com/mountain-pitt/mk-blindcontrol>  
> Wiki: <https://github.com/mountain-pitt/mk-blindcontrol/wiki>

V7/V8 introduced the modern web UI, WiFiManager captive-portal onboarding, LittleFS config storage, Home Assistant auto-discovery, the OTA update endpoint, the dual-servo lockstep mode, and the per-unit trim + slip-correction knobs that make a pair of servos track a single blind reliably. That upstream wiki is still the reference for the hardware pin map, the MQTT command set, HA/openHAB integration examples, and the WiFiManager captive-portal flow — flashing this fork does **not** change any of that, so those docs still apply.

### TR Studios — V9 (this fork)

V9 is a layered **reliability-and-bug-fix pass** on top of V8. All hardware pins, the WiFi config blob, MQTT topics, and HA discovery payloads are unchanged — you can flash V9 over a V8 install and the device keeps its config.

The list of what changed is below. The full per-release breakdown is in [`CHANGELOG-v9.md`](../CHANGELOG-v9.md).

---

## V8 → V9 — the big changes

### Bring-up can no longer hang on a dead broker

V8's `setup()` contained unbounded `while(!WiFi.connect())` and `while(!mqtt.connect())` loops. If either was unreachable, `setup()` never returned — the chip stayed alive, responded to ping, but the web UI never started. You couldn't fix a bad broker address without a reflash. V9.2 bounds both loops at 30 seconds and falls through to `httpServer.begin()` regardless. `loop()`'s existing background reconnect logic handles the broker coming back later. **The web UI is always reachable, even when the broker is down.**

### Non-blocking servo state machine

V8 ran servo moves synchronously with multi-second `delay()` calls in SLOW mode. While a move was in progress, MQTT and WiFi housekeeping stalled, and the hardware watchdog could fire during long slow moves. V9.1 replaces the blocking move with a cooperative state machine in `loop()`. Watchdog gets fed between every degree step, MQTT stays responsive mid-move, and move-complete telemetry reports the actual post-move position instead of the stale pre-move one.

### FAST is actually fast again

V9.1 accidentally made `FAST` feel sluggish — its state-machine was stepping degree-by-degree at 5 ms per degree (~900 ms for a full-range move) because `cacheConfig()` didn't recognise the `FAST` speed string and fell through to `MED`. V9.3 fixed both: `FAST` maps to `SPEED_HIGH`, which takes an immediate-write path that matches upstream V8's feel — single `myservo.write(target)`, wait the servo's native travel time (~300 ms full range), done. Still non-blocking — the state machine stays active for the estimated travel time so `servoWaitDone()` continues to work for trim and slip recovery.

### No more killing an in-flight move when the web UI loads

V8 eagerly called `servo.detach()` on `save_state()`, `HomePage()`, and MQTT reconnect — any of which could fire while a move was in progress and abort the move mid-travel. V9.1 guards all three paths with a "move-in-progress?" check.

### Display version honest, config-blob version frozen

The stored config blob records a `software_version` string, and WiFiManager uses it as part of the AP SSID suffix. Bumping that string would invalidate every unit's saved config. V9 keeps `software_version` frozen at `"V8"` for compat but adds a separate `firmware_installed` string that's used everywhere the user sees a version — web UI header, MQTT `sw_version` attribute, HA discovery payload. **Flash over V8 and your config survives; the UI reports `V9.x` correctly.**

### Auto-OTA + firmware-check removed (V9.8)

V9.1–V9.7 tried to keep upstream's pull-OTA + version-check features working against GitHub. BearSSL on the ESP8266 needs 10-14 KB of **contiguous** heap for the TLS handshake against github.com. Under live operating load (MQTT connected, web UI active, servo state machine running), the unit has around 6 KB of contiguous heap — consistently too little. V9.5 shrunk TLS buffers to 512 bytes. V9.6 reordered page rendering so HTTPS happens before heap is consumed by the page string. V9.7 refactored the URL config to a single editable base. None of it solved the underlying constraint. The honest answer: **the ESP8266 is not a reliable HTTPS client under memory pressure**, so V9.8 removes Auto and Check entirely. The `Releases URL` field on the Setup page now just populates a plain external link on the Firmware page — click it to open the release listing in your browser, download the `.bin`, then use the Manual upload button to flash it. If you maintain your own fork, set that field to your own releases page.

### Smaller housekeeping wins

- **Watchdog-safe wait**: `servoWaitDone()` uses signed-elapsed arithmetic, so a 49-day `millis()` rollover mid-move doesn't wedge the chip.
- **String churn out of the hot path**: `loop()` and `tele_update()` pre-parse config once (`cacheConfig()`) into typed enums/bools/ints, instead of doing string comparisons every tick.
- **`volatile` on shared servo state**: fields read from both `loop()` and async callbacks are marked `volatile` so the compiler can't cache stale values.
- **WiFiManager 0.16 → 2.0.17**: current upstream stable.
- **MQTT buffer 1200 → 2048 B**: HA discovery payloads no longer truncate on some unit configurations.

## Compatibility

| Area | V9 vs upstream V8 |
|---|---|
| GPIO pin map (servo 0 → 13/D7, servo 1 → 14/D5, state-read pin 0) | **Unchanged** |
| Dual-motor lockstep (both written together, servo 0 reads state back) | **Unchanged** |
| WiFi config blob (`/V8.json` in LittleFS, `MK-BlindsControl-V8-<chipid>` AP SSID) | **Unchanged** |
| MQTT topic layout (`cmnd/<id>/POWER`, `stat/<id>/*`, `tele/<id>/LWT`) | **Unchanged** |
| HA discovery (`homeassistant/cover/<id>/config`, retained) | **Unchanged** |
| Manual OTA endpoint (`/firmware`, HTTP POST `.bin`, auth-guarded) | **Unchanged** |
| Pull-OTA + version-check (`/firmwareauto`, `/firmwarecheck`) | **Removed in V9.8** |

Rolling back to V8 is flash-and-go — nothing in V9 touched the config schema.

## Getting the firmware

Always-current binary:

```
https://github.com/TehRobot-Assistant/mk-blindcontrol/releases/latest/download/mk-blindcontrol.bin
```

Or pick a specific version from the [Releases page](https://github.com/TehRobot-Assistant/mk-blindcontrol/releases).

Flash with NodeMCU Flasher, `esptool.py`, or PlatformIO — ESP-12E settings are 4 MB flash, DIO mode, 40 MHz, write to `0x00000`. There's a full flashing walkthrough in [`build/FLASH.md`](../build/FLASH.md).

## Credits

- **Matt Kaczynski / MK Smarthouse** — original hardware + firmware + the mechanical cleverness that makes the whole thing work. <https://www.mksmarthouse.com/>
- **mountain-pitt** — V7 + V8 upstream, open-source maintenance, the web UI and HA integration this fork inherited. <https://github.com/mountain-pitt/mk-blindcontrol>
- **TR Studios** — V9 reliability fork. This repo.

MIT licensed (matching upstream).
