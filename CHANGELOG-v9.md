# MK-BlindsControl v9.1 — community reliability patch

This is a drop-in replacement for upstream v8 (`mountain-pitt/mk-blindcontrol`) targeting the same ESP-12E hardware and the same kit. Config compatibility is preserved — your existing `V7.json` / `V8.json` config files are loaded unchanged.

The dual-motor behaviour (two servos driven in lockstep from GPIO 13 and 14) is preserved exactly. Both servos are attached, written, stepped and detached together. There is no single-motor mode.

**v9.1 reflects a four-angle red-team pass** (C++/Arduino correctness, hardware safety, regression vs upstream, ESPHome parity). The v9 draft had real blockers — overflow bugs, ignored watchdog, mid-move detach paths, stale-publish timing. v9.1 fixes those. See § _v9.1 red-team fixes_ below for the itemised list.

---

## Why v9 exists

v8 exhibits recurring problems — MQTT drops during blind movement, slow web UI, and random reboots after a few days of uptime. These all trace to two architectural issues: blocking `delay()` during servo moves, and Arduino `String` allocation in the hot path fragmenting the 30-40 KB heap. v9 fixes both without changing the feature set.

---

## What changed vs upstream v8

### 1. Non-blocking dual-servo state machine

**Upstream v8:** `moveServo()` calls `delay(40)` per 1° step in SLOW mode (up to 7.2 s of blocked CPU for a 180° travel) and `delay(2000)` after every direct position set. During that time `client.loop()`, `httpServer.handleClient()`, and `MDNS.update()` don't run — so MQTT pings miss, Home Assistant sees the device drop off, and the web UI hangs.

**v9:** added a `ServoState` struct + `servoTick()` helper. `moveServo()` now sets target and returns. `loop()` calls `servoTick()` every iteration, which advances both servos by one degree at the configured interval (LOW=80 ms, MED=40 ms, HIGH=5 ms, SLOW=80 ms). Between steps, MQTT / HTTP / mDNS / buttons all run normally. Servos automatically detach 2 s after reaching target to cut idle current draw.

**v9.1 additions:**
- `ServoState` fields marked `volatile` — memory-visibility defence against WiFi async callbacks observing torn reads.
- `servoStateInit()` called explicitly in `setup()` — doesn't rely on the Xtensa toolchain honouring in-class initialisers for globals.
- `pendingPublish` flag on the state struct — set when a move completes; `loop()` consumes it to re-fire `process_state()` + `publish_state()` so HA sees the *post-move* position, not the stale pre-move one.
- `servoDetachBothIfIdle()` replaces unconditional detach at several upstream call sites.

### 2. `servoWaitDone()` for genuinely synchronous callers

Callers that need to wait for the servo to finish before proceeding — pre-restart install-position flow, slip correction, trim adjustment — use `servoWaitDone(timeoutMs)`. While waiting, the function pumps `servoTick()` + `client.loop()` + `httpServer.handleClient()` + `ESP.wdtFeed()` + `yield()` cooperatively.

**v9.1 fixes:**
- **millis() overflow bug fixed.** v9 used `deadline = millis() + timeout` + `(long)(now - deadline) < 0`, which returns instantly after ~49 days. Replaced with elapsed-time arithmetic: `start = millis(); (millis() - start) < timeout`.
- **Hardware watchdog fed explicitly.** v9 only called `yield()`, which feeds the *soft* watchdog. The *hardware* watchdog (8 s default) would reset the chip during a 15 s SLOW travel. v9.1 calls `ESP.wdtFeed()` every loop iteration.
- **MQTT keep-alive actually serviced during the wait** — v9 comment claimed this but `yield()` doesn't invoke user-level callbacks. v9.1 explicitly calls `client.loop()` and `httpServer.handleClient()` inside the wait.

### 3. Typed cached config

**Upstream v8:** hot paths do `String(blinds_speed).equalsIgnoreCase("LOW")` — four of these in `moveServo()`, plus more in `loop()`. Each call allocates, copies and frees an Arduino `String` on the ~30 KB ESP8266 heap. Days of uptime fragment the heap and cause malloc failures that present as reboots or MQTT disconnects.

**v9:** `cacheConfig()` parses char[] config into typed enums / bools / ints exactly once per config load.

**v9.1 fix:** `cacheConfig()` is now called unconditionally after the WiFiManager portal writes its custom-parameter values back to the char[] buffers — previously only called in the `if (shouldSaveConfig)` branch, which meant a portal run that didn't persist left the cached enums stale.

### 4. Bugfix: `blinddelay` was computed but never applied

**Upstream v8:** `moveServo()` computes `blinddelay` as 80/40/0 for LOW/MED/HIGH — but the step loop uses `delay(40)` hardcoded. All four speeds were effectively identical.

**v9:** step interval derived from `cachedSpeed` in the state machine: 80 ms / 40 ms / 5 ms / 80 ms for LOW / MED / HIGH / SLOW. All four speeds now actually differ.

### 5. WiFi watchdog

**Upstream v8:** only reconnects MQTT if `client.connected()` is false. If WiFi itself drops (AP reboot, channel change, brief roaming), MQTT can never reconnect because the TCP layer has no route.

**v9:** `loop()` monitors `WiFi.status()`. If disconnected for >30 s, calls `ESP.restart()`. MQTT reconnect attempts rate-limited to one every 5 s.

### 6. MQTT buffer bumped 1200 → 2048

**Upstream v8:** `MQTTClient client(1200)`. Home Assistant auto-discovery payloads with dual-servo + battery + state + availability topics can exceed 1200 bytes, causing silent publish drops.

**v9:** `MQTTClient client(2048)`.

**v9.1 correction:** removed the misleading `MQTT_MAX_PACKET_SIZE` build flag from `platformio.ini`. That flag is `PubSubClient`-specific and has zero effect on 256dpi/arduino-mqtt, which is purely constructor-configured.

### 7. Detach-before-blocking-connect

**Upstream v8:** `connect()` contains `while (!client.connect(...)) { delay(1000); }`. If the MQTT broker is down on reboot, servos may remain attached for minutes, drawing 400+ mA on a shared 500 mA regulator.

**v9.1:** `connect()` calls `servoDetachBoth()` before entering the blocking loops. Each blocking loop also calls `ESP.wdtFeed()` so the hardware watchdog doesn't reset the chip during extended broker-down windows.

### 8. Non-blocking install-position flow

**Upstream v8:** `delay(20000)` before `ESP.restart()`. Servos continue drawing PWM during that window.

**v9.1:** `servoWaitDone(20000UL)` (widened from the v9 draft's 15 s to leave headroom for full-travel at SLOW speed), followed by `delay(5000)` as the user-facing disconnect window.

### 9. Hot-path raw servo-writes routed through the state machine

**Upstream v8:** `handleServo()` (`/setPOS` HTTP endpoint) writes directly to `myservo[0]` / `myservo[1]` with `delay(15)` between. If the servos are currently detached, these writes hit the void and the state machine's `current` field drifts out of sync.

**v9.1:** `handleServo()` routes through `moveServo(pos)` so the state machine stays coherent.

### 10. Guarded detach at `save_state()` and `HomePage()`

**Upstream v8:** both functions unconditionally detach both servos. With the non-blocking state machine, these calls frequently hit while a move is in progress — which cut PWM mid-travel and left the blind stuck at an arbitrary angle.

**v9.1:** both call `servoDetachBothIfIdle()` which is a no-op when `servoState.active == true`.

### 11. Trim adjustment + slip correction cooperative waits

**Upstream v8:** calls `moveServo(0)` then immediately `moveServo(trimValue)` or `moveServo(180)` → `moveServo(target)`. Upstream relied on blocking moveServo to complete the first move before the second overwrote target.

**v9.1:** `servoWaitDone(20000UL)` inserted between the alignment/open move and the follow-on move so the mechanical slack compensation and trim offset actually happen.

### 12. Config/ABI compatibility

No changes to `config.json` schema. No changes to web UI field names. No changes to MQTT topic structure. A device running v8 can be OTA-updated to v9.1 and keeps its existing configuration.

Servos keep their GPIO pins (13, 14), keep their pulse-width calibration (544–2200 µs), keep their startup home position (180°).

### 13. Library version bumps

| Library | v8 | v9.1 |
|---|---|---|
| `tzapu/WiFiManager` | `0.16.0` (2020) | `^2.0.17` (current) |
| `bblanchon/ArduinoJson` | `^6.18.5` | `^6.21.5` |

MQTT client (`256dpi/MQTT`) and `HAMqttDevice` unchanged.

---

## v9.1 red-team fixes (summary)

Findings from four independent review angles on the v9 draft:

| # | Angle | Finding | v9.1 fix |
|---|---|---|---|
| 1 | C++ | `servoState` global init not guaranteed before `setup()` on Xtensa | Explicit `servoStateInit()` in `setup()` |
| 2 | C++ | `servoWaitDone()` millis() overflow returns instantly ~49 days | Elapsed-time arithmetic |
| 3 | C++ | 15s tight loop exceeds 8s hardware WDT | `ESP.wdtFeed()` every iteration |
| 4 | C++ | `MQTT_MAX_PACKET_SIZE` build flag is PubSubClient-specific | Removed from platformio.ini |
| 5 | C++ | `cacheConfig()` not called in non-save portal path | Called unconditionally after portal strcpy block |
| 6 | C++ | `ServoState` not `volatile` | Fields marked `volatile` |
| 7 | C++ | Removing `delay(10)` may stall 256dpi MQTT | Retained tiny `delay(1)` after `client.loop()` |
| 8 | Hardware | `handleServo()` raw writes bypass state machine | Routes through `moveServo()` |
| 9 | Hardware | `connect()` blocking loop while servos attached | Detach before loop, `ESP.wdtFeed()` inside |
| 10 | Hardware | `save_state()` detaches mid-move | `servoDetachBothIfIdle()` |
| 11 | Hardware | `HomePage()` detaches on every page load | `servoDetachBothIfIdle()` |
| 12 | Hardware | `setup()` detaches before initial move finishes | `servoBeginMove(ServoPos) + servoWaitDone()` |
| 13 | Regression | `publish_state` fires with stale position | `pendingPublish` flag, deferred publish in `loop()` |
| 14 | Regression | Trim adjustment second-move overwrites first | `servoWaitDone(20000UL)` between |
| 15 | Regression | Install flow 15s edge case at SLOW speed | Widened to 20s |
| 16 | ESPHome | `min_level`/`max_level`/`restore` not valid servo keys | Moved to output `min_power`/`max_power`; use `restore: true` |
| 17 | ESPHome | Inversion branches swapped vs upstream semantics | Branches corrected |
| 18 | ESPHome | `App.safe_mode_manager().request_safe_mode()` doesn't exist | `App.request_safe_mode()` |
| 19 | ESPHome | `on_boot` servo_attach double-writes with restore | Removed; `restore: true` is sufficient |
| 20 | ESPHome | `auto_detach_time: 2s` too short on stiff mechanisms | 5s |
| 21 | ESPHome | Battery multiply 4.86 inherited from raw-count formula | Corrected to 4.97, calibration note added |
| 22 | ESPHome | OTA reflash leaves servos attached | `on_shutdown` detach hook |
| 23 | ESPHome | `frequency: 50 Hz` (with space) parsing flaky | Changed to `50Hz` |

Deliberately deferred to v10:
- **Stall detection.** No back-EMF or current sensing on the hardware, so detection must be heuristic (commanded position not tracking over N ticks). Worth shipping as a separate feature once tested against real jam scenarios.
- **Battery undervoltage lockout.** Low-severity — relevant only for battery-only installs. Publishing a low-battery MQTT alert and auto-detaching below 3.3 V would be ~20 lines, but not critical.

---

## What did NOT change

- Web UI HTML (`PageIndex.h`, `CSS.h`) — identical to upstream.
- MQTT topic structure.
- Home Assistant auto-discovery payload shape (only the packet buffer size).
- OpenHAB integration files (`.items`, `.sitemap`, `.things`).
- OTA update URL.
- Pin assignments.

---

## Upgrade path

1. Flash v9.1 firmware (via OTA or serial) to a device currently running v8.
2. On boot, the existing v8 `.json` config file is loaded and re-saved as v9 (same schema).
3. No reconfiguration needed.
4. First restart takes ~5 s; no longer 20 s.

**Rollback:** flash v8 `.bin` back. Config is forward- and backward-compatible.

---

## Upstream contribution

This diff is suitable for a single PR against `mountain-pitt/mk-blindcontrol#main`. If upstream prefers a staged merge, the logical order is:

1. Bump MQTT buffer + WiFiManager (lowest-risk, highest immediate benefit)
2. Add `cacheConfig()` + typed enums + rewrite hot-path String comparisons
3. Add servo state machine + `servoWaitDone()` + non-blocking `moveServo()` + all the v9.1 hardening
4. Add WiFi watchdog + detach-before-connect

Stages 1-2 alone deliver most of the stability gains.
