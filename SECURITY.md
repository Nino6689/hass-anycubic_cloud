# Security policy

This is a small community fork of [WaresWichall/hass-anycubic_cloud](https://github.com/WaresWichall/hass-anycubic_cloud) — scope is limited to compatibility patches needed to keep the integration running on current Home Assistant. We don't audit Anycubic's own cloud API or MQTT protocol.

## Reporting a vulnerability

For issues specific to this fork's patches, please open a regular issue (or a private security advisory via the **Security** tab if it's sensitive).

For vulnerabilities in the integration's overall behaviour (token handling, MQTT, cloud API auth flow, etc.), please report upstream at [WaresWichall/hass-anycubic_cloud/issues](https://github.com/WaresWichall/hass-anycubic_cloud/issues) — those fixes belong in upstream and we'll mirror them here.

## Tokens

Your slicer / web auth token is stored by Home Assistant in `.storage/core.config_entries` on the HA host (same place as every other integration credential). Treat that file as sensitive — don't paste its contents into issue reports.
