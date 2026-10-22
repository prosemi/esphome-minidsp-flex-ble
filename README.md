# esphome-minidsp-flex-ble

ESPHome Bluetooth LE bridge for miniDSP Flex: control volume, mute, input, preset and
Dirac Live from Home Assistant.

An ESP32 keeps a permanent Bluetooth LE connection to the Flex and exposes it to
Home Assistant as normal entities. Use them from dashboards, automations, voice
assistants or physical remotes (e.g. a Zigbee rotary dial) – no cloud, no phone app.

## KD-1 alternative – with Home Assistant

The bridge does over Bluetooth LE what the miniDSP KD-1 knob does for the Flex, but it
is **not a standalone remote**. The ESP32 has no knob or screen of its own – it only
connects the Flex to Home Assistant. You control the Flex from whatever Home Assistant
offers:

- a dashboard (sliders, buttons, tiles),
- a physical remote, e.g. a **Philips Hue Tap Dial** paired with Zigbee2MQTT or ZHA,
- automations, scenes and voice assistants.

**What you need**

- miniDSP Flex with Bluetooth LE (see [Compatibility](#compatibility))
- an ESP32 board with Bluetooth LE (e.g. M5Stack AtomS3 Lite)
- Home Assistant with ESPHome
- optional: a physical remote such as the Philips Hue Tap Dial

Compared to the KD-1, a physical remote goes through more hops (remote → Zigbee →
Home Assistant → ESP32 → Flex), so expect a short delay instead of an instant
response. In return the Flex becomes part of your smart home: volume in automations,
on any dashboard, from any remote you already have.

The KD-1 and the bridge cannot be connected at the same time – the Flex accepts only
one Bluetooth LE client.

> Not affiliated with or endorsed by miniDSP. Use at your own risk: a wrong volume
> command can be loud. Keep `max_db` conservative and test with the amplifier off or
> the Flex muted first.

## Features

- Volume (absolute, 0.5 dB steps) and **relative volume action** for remotes and buttons
- Mute, Dirac Live on/off, input selection, presets 1–4
- All states are read from the Flex – changes made with the IR remote or Device
  Console show up in Home Assistant too
- Safe by design: volume limit (`max_db`, default −15 dB), at most +2 dB per step and
  at most 4 dB above the last confirmed volume, no unmute above the limit
- One command at a time, the next one only after the Flex confirms – fast turns of a
  dial are merged, never lost or replayed
- Reconnects on its own after a power cut or when the Flex comes back in range

## Compatibility

The bridge talks to the same Bluetooth LE control channel that the miniDSP KD-1 knob
uses, so it targets the Bluetooth-equipped Flex models the KD-1 supports over BLE:

| Model | Status |
|---|---|
| Flex (2x4), Flex DL | ✅ tested |
| Flex Eight, Flex Eight DL | ⚠️ expected to work, not tested yet – reports welcome |
| Flex HT, Flex HTx | ❌ no Bluetooth LE |
| SHD series, Tide16 | ❌ controlled over Wi-Fi, not BLE |

- The Flex must show **BLE** under its name in miniDSP Device Console. Older units may
  need a firmware update first.
- The input list matches the Flex 2x4. Dirac Live entities only make sense on DL models.
- **The Flex accepts one BLE client at a time.** A KD-1 or a phone app connected over
  Bluetooth will keep the bridge from connecting.

**ESP32 board:** tested on an M5Stack AtomS3 Lite (ESP32-S3) with ESPHome 2026.1. Any
ESP32 with Bluetooth LE should work – adjust the `esp32:` section. Place it within
good Bluetooth range of the Flex.

## Installation

1. **Find the Bluetooth MAC address of your Flex**, e.g. with
   [nRF Connect](https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-mobile)
   on your phone. Disconnect the phone afterwards.
2. In the ESPHome dashboard create a new device, then replace its YAML with
   [`minidsp-flex-ble.yaml`](minidsp-flex-ble.yaml).
3. Add the entries from [`secrets.yaml.example`](secrets.yaml.example) to your ESPHome
   `secrets.yaml` (Wi-Fi, API key, OTA password, Flex MAC).
4. If you don't use an AtomS3 Lite, change the `esp32:` section for your board.
5. **Install** (first time over USB, later wirelessly).
6. Home Assistant discovers the device – accept it under *Settings → Devices &
   services*. The entities appear immediately.

Optional settings in `substitutions:` at the top of the YAML:

| Key | Default | Meaning |
|---|---|---|
| `name` | `minidsp-flex-ble` | Device name. **Changes all entity IDs and the action name.** |
| `friendly_name` | `miniDSP Flex BLE` | Display name in Home Assistant |
| `max_db` | `-15.0` | Loudest volume allowed from Home Assistant |

## Entities

With the default `name` (`minidsp-flex-ble`):

| Entity | What it does |
|---|---|
| `number.minidsp_flex_ble_flex_volume_control` | Set the volume in dB |
| `sensor.minidsp_flex_ble_flex_volume` | Current volume in dB |
| `switch.minidsp_flex_ble_flex_mute_control` | Mute on/off |
| `binary_sensor.minidsp_flex_ble_flex_muted` | Mute state |
| `switch.minidsp_flex_ble_flex_dirac_control` | Dirac Live on/off |
| `binary_sensor.minidsp_flex_ble_flex_dirac_live` | Dirac Live state |
| `select.minidsp_flex_ble_flex_input_control` | Input: Analog, Toslink, SPDIF, USB, Bluetooth |
| `sensor.minidsp_flex_ble_flex_input` | Current input |
| `select.minidsp_flex_ble_flex_preset_control` | Preset 1–4 |
| `button.minidsp_flex_ble_flex_refresh_status` | Re-read all states from the Flex |
| `binary_sensor.minidsp_flex_ble_flex_ble_connected` | BLE link to the Flex |
| `binary_sensor.minidsp_flex_ble_flex_control_ready` | Connected and volume known – use it as a condition |
| `sensor.minidsp_flex_ble_flex_control_result` | Last status / error message (diagnostic) |

## Relative volume (remotes, buttons)

The device provides the action **`esphome.minidsp_flex_ble_flex_volume_delta`**:

```yaml
action: esphome.minidsp_flex_ble_flex_volume_delta
data:
  delta_db: 1.0    # positive = louder, negative = quieter, at most ±2 dB per call
```

Calls are accumulated on the ESP, so you can send them as fast as the remote
produces events. Examples:

- [`examples/automation-hue-tap-dial-z2m.yaml`](examples/automation-hue-tap-dial-z2m.yaml) –
  Philips Hue Tap Dial via Zigbee2MQTT, 0.5–2 dB per event depending on rotation speed
- [`examples/automation-volume-buttons.yaml`](examples/automation-volume-buttons.yaml) –
  any two buttons, 1 dB per press

## Protocol

The Flex exposes GATT service `1F00` with characteristic `2F00` (write + notify).
Commands are ASCII text `CMD <hex>\r\n`, replies are ASCII hex in notifications. The
values can be observed with any BLE explorer such as nRF Connect.

| | Command | Reply |
|---|---|---|
| Read volume + mute | `0505FFDA02` | `0605FFDA <volume> <mute>` |
| Read input | `0505FFD901` | `0505FFD9 <input 0–4>` |
| Read preset | `0505FFD801` | `0505FFD8 <preset 0–3>` |
| Read Dirac Live | `0505FFE001` | `0505FFE0 <0 on, 1 off>` |
| Set volume | `0342<v>` | `02AB` (ack) |
| Set mute | `0317<0/1>` | `02AB` |
| Set input | `0334<0–4>` | `02AB` |
| Set preset | `0425<0–3>01` | `02AB` |
| Set Dirac Live | `033F<0 on / 1 off>` | `02AB` |

Volume: `v = −2 × dB` (higher value = quieter, `FF` = −127.5 dB).
Inputs: 0 Analog, 1 Toslink, 2 SPDIF, 3 USB, 4 Bluetooth.

## Troubleshooting

- **`Flex BLE connected` stays off** – check the MAC address, make sure no phone app or
  KD-1 is connected to the Flex over Bluetooth, move the ESP closer, and confirm the
  Flex shows *BLE* in Device Console.
- **Connected but nothing happens** – set `logger: level: DEBUG` and watch the ESPHome
  logs: every command (`TX`) and reply (`RX`) is printed with the `flex` tag.
- **Volume does not go above some value** – that is `max_db`; raise it in
  `substitutions` only if you are sure.
- **Unmute rejected** – the volume was set louder than `max_db` elsewhere (IR remote,
  Device Console). Lower it first.

## License

[MIT](LICENSE)
