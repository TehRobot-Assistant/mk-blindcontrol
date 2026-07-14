# MK-BlindsControl v10.0 — security + reliability release

Drop-in for v9.x. Same ESP-12E hardware, same dual-servo lockstep behaviour, same config files (`V8.json`) loaded unchanged — no reconfiguration needed on upgrade.

v10 is a security-and-correctness pass on top of v9's reliability work. It closes the unauthenticated-LAN-access surface, stops leaking credentials, fixes a guaranteed long-run crash, and corrects several logic bugs. It also makes the Home Assistant cover fully position-aware.

## Security

- **Authentication on every state-changing endpoint.** New `requireAuth()` helper (HTTP basic auth against the admin credentials) now guards config write/save, file upload/delete/download, the file manager and directory listing, all servo/limit moves, device reboot, and the `/resetcmd` factory-reset route (previously an unauthenticated filesystem wipe — now behind an authenticated `resetcmdHttp()` wrapper). Previously only `/firmware`, `/reset` and `/update` were protected.
- **Credentials no longer echoed in cleartext.** The `/setup` page showed the MQTT and admin passwords verbatim; they are now masked.
- **XSS hardening.** New `htmlEncode()` applied to all config values echoed into the setup and home pages (hostname, MQTT topic/server/port/username, OTA path, identifier).
- **Path-traversal guard.** File download/delete reject any filename containing `..` or `/`.

## Bug fixes

- **`/setup` was world-readable** — now authenticated.
- **Physical button did not update reported state.** Three `HA_Blind_State == "…"` no-op comparisons in the click / double-click handlers were meant to be assignments; Home Assistant now reflects the correct state after a physical press.
- **millis() 49-day rollover** in the telemetry and battery timers (broken despite the v9 banner) — both now use elapsed-time subtraction.
- **Battery percentage** read one bucket past the match (and off the end of the table at full charge); corrected with an explicit 0–100 clamp.
- **Duplicate OTA registration** — a bare, unauthenticated `httpUpdater.setup()` call was removed; only the credentialed registration remains.
- **MQTT Port field was misnamed** `input_host` in the config form, so changing the port silently corrupted the hostname and never updated the port. Fixed.
- **`strcpy` → `strlcpy`** for HTTP-argument and WiFiManager value copies into fixed-size buffers, removing a buffer-overflow path.

## Home Assistant

- Discovery payload now includes a **`device` block** (identifiers, name, manufacturer, model, sw_version) so entities group correctly in the device registry instead of appearing orphaned.
- Added **`position_topic` / `set_position_topic`** so the HA cover card's position slider works end-to-end (percentage round-trips through publish/receive), not just open/close/stop.
- Removed the dead legacy `HAMDiscovery()` code path.

## Housekeeping

- Socket-leak fix: `SSDP.begin()` was being called on every telemetry tick (~60 s), leaking a UDP socket handle each time — a guaranteed multi-day crash. It is now called only once at boot.
