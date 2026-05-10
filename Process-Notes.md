# Tuya TS0601 Garage Door Controller - ZHA Integration

## Process Document

**Device:** Merlin electric roller door with Tuya Zigbee garage door controller  
**Zigbee Model:** TS0601  
**Manufacturer String:** `_TZE204_nklqjk62`  
**Environment:** Home Assistant Container (Docker Compose), ZHA integration, Sonoff Zigbee dongle (EZSP)

-----

## Background and Problem

The Tuya Zigbee garage door controller pairs with ZHA successfully but exposes no useful entities by default. The root cause is a mismatch between the device’s manufacturer string (`_TZE204_nklqjk62`) and what the built-in ZHA quirk expects, combined with the built-in quirk using an older v1 structure that does not map to HA entities via ZHA’s discovery mechanism.

The solution requires:

1. A custom ZHA quirk (v2 builder style) that correctly maps the device’s datapoints to HA entities
1. Suppressing the built-in quirk that otherwise matches and shadows the custom one
1. Making the suppression persistent across container updates

-----

## Prerequisites

- Home Assistant running as a Docker container via Docker Compose
- ZHA integration active with a Zigbee coordinator (EZSP-based, e.g. Sonoff Zigbee dongle)
- SSH access to the Docker host
- The garage door controller paired to ZHA (even if showing no useful entities)

-----

## Step 1 - Enable Custom Quirks Path in HA

Edit `~/homeassistant/config/configuration.yaml` and add the following block. If `nano` fails due to a Ghostty terminal error, prefix with `TERM=xterm-256color`.

```bash
TERM=xterm-256color nano ~/homeassistant/config/configuration.yaml
```

Add to the end of the file:

```yaml
zha:
  custom_quirks_path: /config/custom_zha_quirks
```

Note: Inside the container, the config directory is mounted as `/config`, which maps to `~/homeassistant/config` on the host.

Create the custom quirks directory:

```bash
mkdir ~/homeassistant/config/custom_zha_quirks
```

-----

## Step 2 - Create the Custom Quirk File

Create the following file at `~/homeassistant/config/custom_zha_quirks/ts0601_garage_nklqjk62.py`:

```python
# ============================================================
# File    : ts0601_garage_nklqjk62.py
# Purpose : ZHA v2 custom quirk for Tuya TS0601 garage door controller
#           Manufacturer: _TZE204_nklqjk62
#           zha-quirks version: 1.2.0+
#
# Prerequisites:
#   - Home Assistant with ZHA integration
#   - zha: custom_quirks_path: /config/custom_zha_quirks
#     set in configuration.yaml
#
# Usage   : Place in /config/custom_zha_quirks/
#           Restart Home Assistant, then reconfigure the device in ZHA
#
# Exposes : - switch (DP 1) : trigger the door relay
#           - binary_sensor (DP 3) : door open/closed from hall sensor
# ============================================================

from zhaquirks.tuya.builder import TuyaQuirkBuilder
from zigpy.quirks.v2.homeassistant import EntityType
from zigpy.quirks.v2.homeassistant.binary_sensor import BinarySensorDeviceClass

(
    TuyaQuirkBuilder("_TZE204_nklqjk62", "TS0601")
    .tuya_switch(
        dp_id=1,
        attribute_name="garage_door_trigger",
        translation_key="garage_door_trigger",
        fallback_name="Garage Door Trigger",
        entity_type=EntityType.STANDARD,
    )
    .tuya_binary_sensor(
        dp_id=3,
        attribute_name="contact",
        translation_key="contact",
        fallback_name="Door Sensor",
        device_class=BinarySensorDeviceClass.DOOR,
        entity_type=EntityType.STANDARD,
    )
    .skip_configuration()
    .add_to_registry()
)
```

Key notes on this quirk:

- `TuyaQuirkBuilder` takes `(model, manufacturer)` in that order - i.e. the manufacturer string first, then `TS0601`
- `entity_type=EntityType.STANDARD` is required on both entries; without it they default to `CONFIG`/`DIAGNOSTIC` and are disabled by default in HA
- `fallback_name` is required in zha-quirks 1.2.0+
- DP 1 is the door trigger relay; DP 3 is the hall/magnet sensor

-----

## Step 3 - Suppress the Built-in Quirk (Persistent)

The built-in `ts0601_garage.py` in the zha-quirks package also matches `_TZE204_nklqjk62` and takes priority over custom quirks, preventing the custom one from applying. The fix is to comment out that manufacturer string from the built-in file on every container start.

This is done via a custom `entrypoint` in `docker-compose.yml`. The `find` command locates the file regardless of Python version changes in future images.

Edit `~/homeassistant/docker-compose.yml` to match the following:

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - /home/torsten/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    restart: unless-stopped
    privileged: true
    network_mode: host
    entrypoint: >
      sh -c "
      GARAGE_FILE=$$(find /usr/local/lib -name 'ts0601_garage.py' -path '*/zhaquirks/*' 2>/dev/null | head -1) &&
      if [ -n \"$$GARAGE_FILE\" ]; then
        sed -i 's/.*\"_TZE204_nklqjk62\", \"TS0601\".*/            # (\"_TZE204_nklqjk62\", \"TS0601\"),  # removed - handled by custom quirk/' \"$$GARAGE_FILE\" &&
        echo \"Patched $$GARAGE_FILE\";
      fi &&
      exec python3 -m homeassistant --config /config
      "
```

Apply the change:

```bash
cd ~/homeassistant
sudo docker compose up -d
```

Verify the patch applied on startup:

```bash
sudo docker compose logs homeassistant | grep -i patched
```

Expected output:

```
homeassistant  | Patched /usr/local/lib/python3.14/site-packages/zhaquirks/tuya/ts0601_garage.py
```

-----

## Step 4 - Re-pair the Device

1. In HA, go to **Settings > Devices & Services > ZHA**
1. Remove the garage door controller device entirely
1. Restart Home Assistant: `sudo docker compose restart`
1. Re-add the device via **Add Device** in ZHA
1. After pairing, the device page should show:
- `Quirk: zigpy.quirks.v2.CustomDeviceV2`
- A **switch** entity (Garage Door Trigger)
- A **binary sensor** entity (Door Sensor)

-----

## Step 5 - Momentary Pulse Automation

The trigger switch entity is latching by default - it holds ON until explicitly turned OFF. The Merlin door controller expects a momentary pulse. This automation turns the switch back off 500ms after it is turned on, from any source (HA UI, iOS Home, Siri, etc.).

In HA go to **Settings > Automations > Create Automation > Edit in YAML** and paste:

```yaml
alias: Garage Door - Momentary Trigger Pulse
description: Sends a 0.5s pulse to the garage door trigger switch
triggers:
  - trigger: state
    entity_id:
      - switch.garage_door_garage_door_trigger
    from:
      - "off"
    to:
      - "on"
conditions: []
actions:
  - delay:
      milliseconds: 500
  - target:
      entity_id: switch.garage_door_garage_door_trigger
    action: switch.turn_off
    data: {}
mode: single
```

Note: Verify the entity ID matches your installation. It may differ if you named the device differently during pairing.

-----

## Datapoints Reference

The device reports the following Tuya datapoints (DPs). Only DPs 1 and 3 are mapped to HA entities by the custom quirk; the others are present in the device protocol but not used.

|DP|Type  |Function                        |Mapped             |
|--|------|--------------------------------|-------------------|
|1 |Bool  |Door relay trigger              |Yes - switch entity|
|2 |uint32|Unknown                         |No                 |
|3 |Bool  |Hall/magnet sensor (open/closed)|Yes - binary sensor|
|4 |uint32|Unknown (value: 10)             |No                 |
|5 |uint32|Unknown (value: 3600)           |No                 |
|11|Bool  |Unknown                         |No                 |
|12|enum8 |Unknown                         |No                 |

-----

## Troubleshooting

**Custom quirk loads but device shows no quirk applied**  
The built-in quirk is still winning. Verify the `entrypoint` patch in `docker-compose.yml` is correct and that the startup log shows `Patched ...`. Re-run `sudo docker compose up -d` to force the entrypoint to re-apply.

**Import error on custom quirk load**  
Check the zha-quirks version: `sudo docker exec homeassistant pip show zha-quirks`. The quirk file above is written for version 1.2.0+. Earlier versions used different import paths and parameter names (`dp` instead of `dp_id`, `ContactSwitchCluster` which no longer exists, etc.).

**Device shows `Quirk: zhaquirks.tuya.ts0601_garage.TuyaGarageSwitchTO` but no entities**  
This was the state with older zha-quirks versions where the built-in v1 quirk matched but could not surface entities. The entrypoint patch and custom v2 quirk resolve this.

**Ghostty terminal error with nano**  
Prefix the command with `TERM=xterm-256color`. Optionally add `export TERM=xterm-256color` to `~/.bashrc` on the host to make it permanent.

**docker exec commands hang without sudo**  
The docker socket requires root on this system. Always prefix `docker` commands with `sudo`, and note that subshells in commands like `$(docker ps ...)` also need `sudo` or use the container name directly.