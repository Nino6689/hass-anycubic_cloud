# Anycubic Cloud for Home Assistant

[![Buy Me A Coffee](https://img.shields.io/badge/Buy_me_a_coffee-FFDD00?style=flat-square&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/nino6689)
[![GitHub release](https://img.shields.io/github/v/release/Nino6689/hass-anycubic_cloud?style=flat-square)](https://github.com/Nino6689/hass-anycubic_cloud/releases/latest)
[![hacs](https://img.shields.io/badge/HACS-Custom-41BDF5?style=flat-square&logo=home-assistant&logoColor=white)](https://hacs.xyz)
[![GPLv3](https://img.shields.io/badge/license-GPL_3.0-blue?style=flat-square)](LICENSE)

Real-time monitoring and control of your **Anycubic Kobra / Photon** 3D printer from Home Assistant — temperatures, layer count, progress, ACE multi-colour spool info, live job preview image, pause/resume/cancel, and more.

This is a **community fork** of the original [WaresWichall/hass-anycubic_cloud](https://github.com/WaresWichall/hass-anycubic_cloud) (last upstream release 2024‑12), patched so it works on modern Home Assistant Core (2026.x running Python 3.13/3.14 with paho-mqtt 2.x). Stock v0.2.2 fails to load with `HTTP 500 "Server got itself in trouble"` on current HA — this fork fixes that.

---

## Table of contents

- [Quick start](#quick-start)
- [Supported printers](#supported-printers)
- [Step 1 — Install](#step-1--install-the-integration)
- [Step 2 — Get an auth token](#step-2--get-an-auth-token)
  - [Option A — Slicer (recommended)](#option-a--slicer-token-recommended-mqtt-live-updates)
  - [Option B — Web](#option-b--web-token-easiest-no-live-mqtt)
  - [Option C — Android](#option-c--android-token-advanced)
- [Step 3 — Add it in Home Assistant](#step-3--add-the-integration-in-home-assistant)
- [Step 4 — Choose a Connect Mode](#step-4--choose-a-connect-mode-controls-when-mqtt-is-active)
- [Troubleshooting](#troubleshooting)
- [Frontend card](#frontend-card)
- [What's different from upstream](#whats-different-from-upstream-v022)
- [Support the project](#support-the-project)
- [Credits](#credits)

---

## Quick start

If you already know your way around HACS and dev tools, here's the speed run:

1. **HACS** → ⋮ → **Custom repositories** → add `https://github.com/Nino6689/hass-anycubic_cloud` as an **Integration**.
2. Install **Anycubic Cloud**, restart HA.
3. Grab a token — either the [Slicer one-liner](#option-a--slicer-token-recommended-mqtt-live-updates) (recommended) or the [Web dev-console snippet](#option-b--web-token-easiest-no-live-mqtt).
4. **Settings → Devices & services → Add integration** → search **Anycubic Cloud** → pick your auth mode → paste token.
5. Select your printer(s). Done.

Otherwise, follow the step-by-step below — it's all explained.

---

## Supported printers

Confirmed working (same as upstream):

- **Kobra S1** — including ACE multi-colour spool details
- **Kobra 3 Combo**
- **Kobra 2**, **Kobra 2 Max**, **Kobra 2 Pro**
- **Photon Mono M5s** (basic support)
- **M7 Pro** (basic support)

If you try another model, [open an issue](https://github.com/Nino6689/hass-anycubic_cloud/issues) and let us know if it works — happy to add it to the list.

---

## Step 1 — Install the integration

### Recommended: HACS

1. In Home Assistant, open **HACS**.
2. Click the **⋮ menu** (top right) → **Custom repositories**.
3. Add this repository:
   - **URL**: `https://github.com/Nino6689/hass-anycubic_cloud`
   - **Category**: `Integration`
4. Click **Add**, then close the dialog.
5. Search HACS for **Anycubic Cloud** and click **Download** on this fork.
6. **Restart Home Assistant** (Settings → System → Restart).

### Manual install

If you don't use HACS, grab the latest [release zip](https://github.com/Nino6689/hass-anycubic_cloud/releases/latest), then:

1. Extract the `custom_components/anycubic_cloud/` folder.
2. Copy it into your HA `config/custom_components/` directory.
3. Restart Home Assistant.

---

## Step 2 — Get an auth token

This integration **never asks for your Anycubic email/password directly** — Home Assistant doesn't know how to do the Anycubic OAuth flow with captchas/2FA. Instead you obtain a **token** from somewhere you're already logged in (the slicer, the website, or the Android app) and paste it in.

Pick **one** of the three options below.

### Option A — Slicer token (RECOMMENDED, MQTT live updates)

> ⚡ **Why this one?** Anycubic actively blocks MQTT live updates for tokens obtained from the website. Currently **only tokens from Anycubic Slicer Next** give you real-time printer status (temperatures, progress, layer count) updating every second instead of every minute.

#### What you need

- [**Anycubic Slicer Next**](https://www.anycubic.com/pages/anycubic-slicer-next) installed on your Mac or Windows PC.
- The slicer logged in to your Anycubic account (do this once via **Settings → Account** inside the slicer).

#### Extract the token

After logging in once, **quit the slicer** (so it isn't holding the token in memory) and run one of the commands below — it prints the token to your terminal.

**macOS** — copy/paste into Terminal:

```bash
python3 -c "import json; print(json.load(open(f'{__import__(\"os\").path.expanduser(\"~\")}/Library/Application Support/AnycubicSlicerNext/AnycubicSlicerNext.conf'))['anycubic_cloud']['access_token'])"
```

**Windows** — copy/paste into PowerShell:

```powershell
$conf = "$env:APPDATA\AnycubicSlicerNext\AnycubicSlicerNext.conf"
(Get-Content $conf | ConvertFrom-Json).anycubic_cloud.access_token
```

The token will be a long string (~1200 characters) starting with `eyJ…`. Copy the whole thing.

#### Tidy up (recommended but optional)

After grabbing the token, set `access_token` to `""` in the slicer config file (or log out of the slicer inside its UI). Otherwise both the slicer and HA stay logged in with the same token at the same time, which can knock each other off intermittently.

→ Now jump to [Step 3](#step-3--add-the-integration-in-home-assistant) and pick auth mode **Slicer**.

---

### Option B — Web token (EASIEST, no live MQTT)

> ⚠️ **Trade-off:** Web tokens give you cloud-polling only — printer state updates every ~60 seconds instead of in real time. Fine for "is it printing?" but not for watching the temperature line draw. If you want live monitoring, use Option A instead.

1. Open <https://cloud-universe.anycubic.com/file> in your browser and **sign in**.
2. Open browser **Developer Tools** (F12 on most browsers, or right-click → Inspect).
3. Switch to the **Console** tab.
4. Paste this and press Enter:

   ```js
   window.localStorage["XX-Token"]
   ```

5. The console prints your token as a quoted string. Copy the contents **without the surrounding quotes**.

→ Now jump to [Step 3](#step-3--add-the-integration-in-home-assistant) and pick auth mode **Web**.

---

### Option C — Android token (ADVANCED)

Requires extracting both a token and a `device_id` from the Android app's network traffic (e.g. via [mitmproxy](https://mitmproxy.org/) or [HTTP Toolkit](https://httptoolkit.com/)).

This is significantly more involved than the other two and not really recommended unless you have specific reasons to need it. The HA UI will prompt you for both **User Token** and **Device ID** when you pick the Android option.

---

## Step 3 — Add the integration in Home Assistant

1. **Settings → Devices & services**.
2. Click **+ Add integration** (bottom right).
3. Search for **Anycubic Cloud** and click it.
4. Pick the **authentication mode** matching the token you grabbed in Step 2 (Slicer / Web / Android).
5. Paste your token into the **User Token** box (and **Device ID** if you picked Android).
6. Click **Submit**.
7. Select which of your printer(s) to track — for most people that's just "all of them".
8. Click **Submit** again.

Once HA finishes setting up you'll see a new **Anycubic Cloud** entry in your integrations list, and a new sidebar panel will appear with a printer card.

---

## Step 4 — Choose a Connect Mode (controls when MQTT is active)

MQTT is what gives you sub-second live updates. Connecting all the time puts a tiny bit of extra load on Anycubic's cloud broker, so the integration lets you choose when it should be on.

To change: **Settings → Devices & services → Anycubic Cloud → Configure**.

| Mode | When MQTT connects |
|---|---|
| **Printing Only** (default) | While a print is running |
| **Printing & Drying** | + while the ACE is drying filament |
| **Device Online** | Whenever the printer is powered on |
| **Always** | All the time — best for live monitoring of an idle printer |
| **Never Connect** | Polled only (closest to "Web auth" behaviour) |

### Force MQTT on right now

If you've just changed mode and want MQTT to reconnect immediately rather than wait for the next print:

1. Toggle `switch.<printer>_manual_mqtt_connection_enabled` → **On**.
2. Press `button.<printer>_refresh_mqtt_connection`.
3. Check `binary_sensor.<printer>_mqtt_connection_active` — should turn **on** within 5–15 seconds.

---

## Troubleshooting

### "Cannot connect" or "Invalid authentication" right after submitting the token

- **Most common cause**: token has a trailing space or newline. Re-copy carefully, especially from the browser console (which adds quotes — strip them).
- **Slicer token specifically**: make sure the slicer was **logged in then quit** before you read the config file. If it was open, the file may contain an old/stale token.

### MQTT never connects (entity stays "off")

- Web tokens **don't support MQTT** — that's by Anycubic's design, not a bug. Switch to a Slicer token (Option A).
- For Slicer tokens: check `binary_sensor.<printer>_printer_online` is **on**. If your printer is off / unplugged / on a different network, MQTT can't reach it.
- The default **Connect Mode** is "Printing Only" — if your printer is idle, MQTT will reconnect when the next print starts. To monitor an idle printer, change the mode to **Always** (see [Step 4](#step-4--choose-a-connect-mode-controls-when-mqtt-is-active)).

### Logs show "Unknown mqtt update, type: light" or "multiColorBox/report unhandled"

These are **harmless** — they're MQTT message types the upstream integration's parser doesn't know about yet (newer firmware features). Real telemetry still flows for everything else. Logged at WARNING level, no action needed.

### "Reauthentication required" notification

Your token expired — they last about 5 months. Re-extract using [Step 2](#step-2--get-an-auth-token) and feed the new token through **Settings → Devices & services → Anycubic Cloud → Reconfigure → Re-Auth**. Entity IDs, history, and your dashboard automations are all preserved.

### Still stuck

[Open an issue](https://github.com/Nino6689/hass-anycubic_cloud/issues/new) and include:

- HA version (Settings → About)
- The relevant log lines (Settings → System → Logs → search **anycubic**)
- Which auth mode you tried

---

## Frontend card

This integration pairs with the [Anycubic card for Home Assistant](https://github.com/WaresWichall/hass-anycubic_card) (also from the upstream author) — a nice rich card showing printer state, progress, ACE spools, and a live preview. Install it separately via HACS as a **Frontend** custom repository.

---

## What's different from upstream v0.2.2

This fork makes only the minimum changes needed to keep the integration running on current Home Assistant. We don't add new features — for those, please raise them upstream first.

| Patch | File | Why |
|---|---|---|
| `paho-mqtt==1.6.1` → `paho-mqtt>=1.6.1` | `manifest.json` | HA Core ships paho-mqtt 2.1.0; strict pin failed to install on Python 3.13/3.14 → config flow crashed with HTTP 500 |
| `Client(client_id=…)` → `Client(CallbackAPIVersion.VERSION1, …)` | `anycubic_cloud_api/api/mqtt.py` | paho-mqtt 2.x requires `CallbackAPIVersion` as the first argument. We pass `VERSION1` so the existing callback signatures keep working without rewriting every callback. Falls back gracefully on paho-mqtt 1.x via try/except |
| Added `accept:` to file selector | `services.yaml` | Hassfest validator now requires it |
| Removed literal URLs from auth-step descriptions | `strings.json`, `translations/en.json` | Hassfest rejects "string should not contain URLs" — the README handles them instead |
| Repo URLs + codeowners | `manifest.json` | `documentation` / `issue_tracker` now point at this fork; `codeowners` includes both upstream + this maintainer |
| Added README for macOS users | this file | Upstream README only documented the Windows slicer path |

---

## Support the project ☕

This integration is and always will be **completely free** — use it, fork it, modify it. GPL-3.0, knock yourself out.

If it's saved you a bunch of time, made your 3D printing setup nicer, or just made you smile when the print preview popped up on your dashboard, you can chuck a coffee (or a beer 🍺) into the tip jar:

[![Buy Me A Coffee](https://img.shields.io/badge/Buy_me_a_coffee_or_a_beer-FFDD00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/nino6689)

Zero pressure — but it does fuel the late-night patching when Anycubic next changes their API 😄

If you'd rather support the original author who did 95% of the work, [@WaresWichall is on the upstream repo](https://github.com/WaresWichall/hass-anycubic_cloud#donations) too.

---

## Credits

- **[@WaresWichall](https://github.com/WaresWichall)** — original integration, all the heavy lifting. Massive thanks. ⭐
- **Compatibility patches** by [@Nino6689](https://github.com/Nino6689).
- Frontend card concept originally adapted from [@dangreco](https://github.com/dangreco)'s threedy.

---

## Licence

[GPL-3.0](LICENSE), matching upstream.
