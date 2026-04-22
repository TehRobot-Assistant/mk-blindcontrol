# mk-blindcontrol — V9 community reliability patch

ESP8266 MQTT servo-motor control for venetian blinds. Home Assistant and openHAB integration, built-in web UI.

This is a **community fork** of [`mountain-pitt/mk-blindcontrol`](https://github.com/mountain-pitt/mk-blindcontrol), carrying three layered reliability patches on top of upstream V8. All hardware pins + WiFi-config format stay compatible with V8 — flash over V8, no re-config needed.

## Get the latest firmware

Always-current binary:

```
https://github.com/TehRobot-Assistant/mk-blindcontrol/releases/latest/download/mk-blindcontrol.bin
```

This URL redirects to whichever release is flagged "latest" on GitHub — bookmark it once, never update your flash script again.

Or grab a specific version from the [Releases page](https://github.com/TehRobot-Assistant/mk-blindcontrol/releases).

Flash with **NodeMCU Flasher** / **esptool.py** / PlatformIO (`pio run -t upload`). ESP-12E settings: 4 MB flash, DIO mode, 40 MHz, write to `0x00000`.

## What V9 fixes

V9 is a layered bug-fix pass, documented fully in [`CHANGELOG-v9.md`](./CHANGELOG-v9.md). Summary:

### V9.1 — reliability patch (post red-team)
- Non-blocking servo state machine — eliminates multi-second `delay()` calls in SLOW mode that stalled MQTT + WiFi housekeeping
- Hardware-watchdog feeds during cooperative waits (prevents 49-day `millis()` overflow + boot-reset on long SLOW moves)
- `servoWaitDone()` uses signed elapsed arithmetic (overflow-safe)
- Guarded detach on `save_state()` / `HomePage()` / MQTT-reconnect paths (no more killing an in-flight move when the web UI is loaded)
- Deferred MQTT-publish on move-complete (reports actual post-move position instead of stale pre-move)
- `volatile` on shared servo-state fields against async callback reads
- String-churn eliminated from `loop()` + `tele_update()` hot path (cached typed config)
- Upgraded WiFiManager 0.16 → 2.0.17 (current stable)
- MQTT packet buffer bumped 1200 → 2048 bytes

### V9.2 — bring-up hang fix
Upstream V8's `connect()` looped forever on an unreachable MQTT broker, keeping the hardware watchdog fed while `setup()` never returned — web UI was therefore unreachable, and you couldn't fix a bad broker address without a reflash. V9.2 bounds both the WiFi and MQTT connect loops at 30 s, then falls through to `httpServer.begin()` anyway. `loop()`'s existing reconnect path keeps retrying in the background. **You can always reach the web UI to fix config, even when the broker is down.**

### V9.3 — fast is fast, labels are honest
- **`FAST` speed now matches upstream V8's feel.** V9.1 stepped every degree at 5 ms/°, which felt sluggish (~900 ms full-range). V9.3 adds an immediate-write path for `SPEED_HIGH` that mirrors upstream's single `myservo.write(target)` + settle — the servo travels at its native MG90S speed (~300 ms full range). Still non-blocking: the state machine stays "active" for an estimated travel time so `servoWaitDone()` still works for trim adjust + slip recovery.
- **`cacheConfig()`** now recognises `FAST` as a speed string (previously fell through to MED silently because the `if` ladder only listed LOW/HIGH/SLOW).
- **Web-UI title + MQTT `sw_version` show `V9.3`** instead of `V8`. Internal `software_version` stays `"V8"` for config-blob compat; display uses a separate string.

## Backwards compatibility

| Area | V9.x vs upstream V8 |
|---|---|
| GPIO pin map (servo 0 → 13/D7, servo 1 → 14/D5, state-read pin 0) | **Unchanged** |
| Dual-motor behaviour (both written in lockstep, only servo 0 reads state back) | **Unchanged** |
| WiFi config blob (`/V8.json` in LittleFS, `MK-BlindsControl-V8-<chipid>` AP SSID) | **Unchanged** |
| MQTT topic layout (`cmnd/<id>/POWER`, `stat/<id>/*`, `tele/<id>/LWT`) | **Unchanged** |
| HA discovery (`homeassistant/cover/<id>/config`, retained) | **Unchanged** |
| OTA endpoint (`/firmware`, HTTP POST `.bin`, auth-guarded) | **Unchanged** |

## Rollback

Flash any upstream V8 binary back. Config survives since we never touched `/V8.json`.

## Contributing

PRs welcome on this fork. Reviewable diffs against upstream V8: the whole V9 story is in three logical changes, see `CHANGELOG-v9.md` and the commit history. Hardware test on real ESP-12E + MG90S servos is needed before any non-trivial merge.

## Credits

- **Matt Kaczynski / MKSMARTHOUSE** — original V1/V2 design, hardware + firmware. [mksmarthouse.com](https://www.mksmarthouse.com/)
- **mountain-pitt** — V7 / V8 upstream. [github.com/mountain-pitt/mk-blindcontrol](https://github.com/mountain-pitt/mk-blindcontrol)
- V9.x reliability patches — this fork.

MIT licensed (matching upstream).
