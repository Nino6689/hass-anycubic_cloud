# Anycubic Cloud — community fork

This is a community fork of [WaresWichall/hass-anycubic_cloud](https://github.com/WaresWichall/hass-anycubic_cloud), patched so the integration works on modern Home Assistant Core (2026.x running Python 3.13/3.14 with paho-mqtt 2.x).

Upstream is excellent but the maintainer moved on to another printer brand back in **December 2024** ([their note](https://github.com/WaresWichall/hass-anycubic_cloud/issues/33)), so stock v0.2.2 returns `HTTP 500 "Server got itself in trouble"` on current HA. This fork patches that.

> **Credit:** all the hard work is [@WaresWichall](https://github.com/WaresWichall)'s. The patches here are tiny — see [What's different](#whats-different-from-upstream-v022) below.

## Supported printers

Same as upstream — confirmed working with Kobra S1 (incl. ACE multi-colour), Kobra 3 Combo, Kobra 2, Kobra 2 Max, Kobra 2 Pro, Photon Mono M5s, M7 Pro. If you have another model and it works (or doesn't), open an issue here.

## What's different from upstream v0.2.2

| Patch | Where | Why |
|---|---|---|
| `paho-mqtt==1.6.1` → `paho-mqtt>=1.6.1` | `manifest.json` | HA Core ships paho-mqtt 2.1.0; the strict pin failed to install on Python 3.13/3.14 → config flow crashed |
| `Client(client_id=…)` → `Client(CallbackAPIVersion.VERSION1, …)` | `anycubic_cloud_api/api/mqtt.py` | paho-mqtt 2.x requires `CallbackAPIVersion` as first arg. `VERSION1` keeps existing callback signatures so nothing else changes. Falls back gracefully on paho-mqtt 1.x |
| Version → `0.2.2-nb1` | `manifest.json` | Marks the patched release |
| **New: Mac slicer-token instructions** | this README | Upstream README only covers Windows. macOS path documented below |

## Install via HACS (custom repository)

1. HACS → ⋮ menu → **Custom repositories**
2. Add: `https://github.com/Nino6689/hass-anycubic_cloud` · Category: **Integration**
3. Find **Anycubic Cloud (community fork)** in the list → Install
4. Restart Home Assistant
5. Settings → Devices & Services → Add Integration → search **Anycubic Cloud**

## Manual install

Copy `custom_components/anycubic_cloud/` into your HA `config/custom_components/` directory and restart.

## Getting an auth token

The integration never accepts your email/password directly — all three auth modes need a pre-extracted token. The slicer path is recommended because **MQTT (live updates) currently only works with slicer tokens** ([upstream note](https://github.com/WaresWichall/hass-anycubic_cloud/issues/33)).

### Option A — Slicer auth (recommended — supports MQTT)

1. Install **Anycubic Slicer Next**, log in to your Anycubic account, and quit it.
2. Extract `access_token` from its config:

   **macOS** (one-liner):
   ```sh
   python3 -c "import json; print(json.load(open(
     '$HOME/Library/Application Support/AnycubicSlicerNext/AnycubicSlicerNext.conf'
   ))['anycubic_cloud']['access_token'])"
   ```

   **Windows** — open `%AppData%\AnycubicSlicerNext\AnycubicSlicerNext.conf` in a text editor, find `"access_token": "..."`, copy the string inside the quotes.

3. In HA: **Add Integration → Anycubic Cloud → Slicer → User Token** (paste).

> Recommended: after grabbing the token, set `access_token` to `""` in the slicer config (or log out of the slicer) so the slicer and HA aren't both logged in at the same time, which can knock each other off.

### Option B — Web auth (no MQTT — 60 s polling only)

1. Log in at <https://cloud-universe.anycubic.com/file>
2. Open browser dev tools (F12) → Console tab
3. Paste:
   ```js
   window.localStorage["XX-Token"]
   ```
4. Copy the returned string (without the surrounding quotes)
5. In HA: **Add Integration → Anycubic Cloud → Web → User Token** (paste)

### Option C — Android auth

Requires extracting token + device_id from the Android app's traffic (mitmproxy / similar). Not covered here — see upstream.

## Connect Modes

After setup, the integration's **Configure** menu offers:

| Mode | When MQTT connects |
|---|---|
| Printing Only (default) | While a job is running |
| Printing & Drying | + while ACE drying |
| Device Online | Whenever printer is online |
| Always | All the time (best for live monitoring; slightly more wear on the cloud MQTT broker) |
| Never Connect | Polled only — closest to "Web auth" behaviour |

To force MQTT on right now regardless of the mode: toggle `switch.<printer>_manual_mqtt_connection_enabled` → On, then press `button.<printer>_refresh_mqtt_connection`. MQTT should connect in 5-15 s; check `binary_sensor.<printer>_mqtt_connection_active`.

## Known noise (harmless)

After MQTT connects you may see warnings like:
```
Anycubic MQTT Message error … Unknown mqtt update, type: light
Anycubic MQTT Message unhandled data in: process_mqtt_update … multiColorBox/report
```

These are message types upstream's data-model code doesn't have parsers for yet (newer firmware features). Real data still flows for everything else.

## Token expiry

Slicer tokens last ~5 months. When they expire HA fires a re-auth notification — repeat the slicer extraction and feed it back through **Devices → Anycubic Cloud → Reconfigure → Re-Auth**.

Entity IDs, history, and dashboard configurations are preserved across re-auths.

## Frontend card

Upstream's [hass-anycubic_card](https://github.com/WaresWichall/hass-anycubic_card) still works fine with this fork — install separately via HACS.

## Contributing

PRs welcome for further breaking-change fixes. This fork's scope is **only** the patches needed to keep v0.2.2 running on current HA — for new features please open them at upstream first.

## Licence

Same as upstream.
