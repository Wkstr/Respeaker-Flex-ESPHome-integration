# reSpeaker Flex ESPHome Integration

ESPHome voice satellite configuration for the [reSpeaker Flex](https://github.com/respeaker/reSpeaker_Flex) XVF3800 4-mic array, with a XIAO ESP32-S3 as the host.

Built on top of [formatBCE's](https://github.com/formatBCE) voice satellite core and XVF3800 I2C component.

## Supported Firmware

| XMOS Firmware Image | Array | I2S Rate | Channels |
|---|---|---|---|
| `respeaker_flex_i2s_c16k2ch_v*.bin` | Circular | 16 kHz | 2 (comms + ASR) |
| `respeaker_flex_i2s_c48k2ch_v*.bin` | Circular | 16 kHz* | 2 (comms + ASR) |
| `respeaker_flex_i2s_l16k2ch_v*.bin` | Linear | 16 kHz | 2 (comms + ASR) |
| `respeaker_flex_i2s_l48k2ch_v*.bin` | Linear | 16 kHz* | 2 (comms + ASR) |

> **\* Note:** Despite the "48k" in the firmware name, the I2S bus actually
> runs at **16 kHz** on all tested Flex firmware images (v1.0.3/v1.0.4).
> This is a known firmware issue — see
> [reSpeaker_Flex issue #2](https://github.com/respeaker/reSpeaker_Flex/issues/2).
> The ESPHome configuration is the same for all images: 16 kHz, 32-bit,
> stereo, I2S secondary.

## Hardware

- reSpeaker Flex (circular or linear 4-mic array)
- XIAO ESP32-S3 (mounted on the Flex board)

### Pin Map (fixed by board layout)

| Signal | XIAO Pin | GPIO |
|---|---|---|
| I2C SDA | D4 | GPIO5 |
| I2C SCL | D5 | GPIO6 |
| I2S LRCLK | D8 | GPIO7 |
| I2S BCLK | D9 | GPIO8 |
| I2S Mic Data | D6 | GPIO43 |
| I2S Speaker Data | D7 | GPIO44 |
| MCLK (XMOS output) | D10 | — (not used by ESP32) |

The XVF3800 is the **I2S master**. The ESP32-S3 runs as **I2S secondary**.
No MCLK input is needed from the ESP32.

## Quick Start

### 1. Flash XMOS Firmware

Download the appropriate firmware from the [reSpeaker Flex releases](https://github.com/respeaker/reSpeaker_Flex/releases) and flash via USB DFU.

### 2. Create ESPHome Configuration

Create a new device in ESPHome Device Builder and paste one of the example configurations:

- **Circular array, 16k firmware:** [`config/flex-c16k2ch-example.yaml`](config/flex-c16k2ch-example.yaml)
- **Circular array, 48k firmware:** [`config/flex-c48k2ch-example.yaml`](config/flex-c48k2ch-example.yaml)

Or reference the packages directly in your own YAML:

```yaml
substitutions:
  name: my-flex-device
  friendly_name: My Flex Assistant
  flex_ref: main

packages:
  flex:
    url: https://github.com/Wkstr/Respeaker-Flex-ESPHome-integration
    ref: ${flex_ref}
    refresh: 1d
    files:
      - packages/base.yaml
      - packages/hardware.yaml
      - packages/voice-assistant.yaml
      - packages/no-leds.yaml

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

api:
  encryption:
    key: !secret api_key

ota:
  - platform: esphome
    id: ota_esphome
    password: !secret ota_password
```

### 3. Flash ESP32 Firmware

- **Wireless:** Install → Wirelessly (if device is already online)
- **USB:** Install → Manual download → Factory image, then flash with esptool:
  ```bash
  esptool --port /dev/ttyACM0 --baud 460800 write-flash 0x0 firmware.factory.bin
  ```

### 4. Verify

After the device comes online in Home Assistant:

1. **DSP Status** = `OK`, **Firmware Version** = `1.0.4`
2. **Assist satellite** is available
3. Play TTS or media through the media player — audio should be clear
4. Say a wake word — the Assist pipeline should activate

## Features

| Feature | Details |
|---|---|
| HA Assist satellite | Full voice pipeline: wake word → STT → TTS |
| On-device wake word | micro_wake_word (okay_nabu, kenobi) |
| Media player | Announcements, music, TTS playback |
| Software volume/mute | ESPHome software volume; no codec register writes |
| Microphone mute | ESPHome `microphone.mute` (zero-filled buffers) |
| DSP route selection | Comms→STT + ASR→wake word, or ASR→both |
| ASR output gain | 1.0–8.0x |
| AGC max gain (comms) | 1–125 |
| Beam lock | Pin AEC beam to wake-word direction (v1.0.4+) |
| XMOS firmware version | Read-only display; no automatic DFU |
| Timers & alarm | Via formatBCE core |
| LED ring | Not present on Flex (stub hooks provided) |

## Architecture

This repository reuses two upstream projects:

| Component | Source | Purpose |
|---|---|---|
| Voice satellite core | [formatBCE/Respeaker-Lite-ESPHome-integration](https://github.com/formatBCE/Respeaker-Lite-ESPHome-integration) | Boot logic, voice pipeline, media player, timers, sounds |
| XVF3800 I2C component | [formatBCE/Respeaker-XVF3800-ESPHome-integration](https://github.com/formatBCE/Respeaker-XVF3800-ESPHome-integration) | `respeaker_xvf3800` C++ component for XMOS I2C communication |

### Key Differences from formatBCE's XVF3800 4-Mic Array

| Aspect | XVF3800 4-Mic Array | reSpeaker Flex |
|---|---|---|
| I2S sample rate | 48 kHz | 16 kHz |
| Decimator | Yes (48k→16k) | No (native 16k) |
| AIC3104 DAC control | ESP32 via I2C | XMOS firmware owns it |
| LED ring | 12-LED ring | None |
| Mute button | Physical button | None |
| Mic mute method | X0D30 GPIO | ESPHome software mute |
| XMOS DFU | Auto-update via ESPHome | Manual (Seeed provides firmware) |
| MCLK | ESP32 provides 12.288 MHz | XMOS outputs 24.576 MHz (not needed by ESP32) |

## Repository Structure

```
├── packages/
│   ├── base.yaml              # Node identity, ESP32 build, I2C, external components
│   ├── hardware.yaml           # Flex I2S pins, mic, speaker, XMOS DSP, controls
│   ├── voice-assistant.yaml    # Beam lock, wake word gain, volume range
│   └── no-leds.yaml            # LED hook stubs (Flex has no LED ring)
├── config/
│   ├── common/
│   │   └── flex-satellite-base.yaml   # All packages combined
│   ├── flex-c16k2ch-example.yaml      # Circular 16k example
│   └── flex-c48k2ch-example.yaml      # Circular 48k example
└── README.md
```

## Troubleshooting

### No microphone audio / full-scale noise

- Confirm the ESP32 is **I2S secondary** (`i2s_mode: secondary`)
- Confirm sample rate is **16000** (not 48000)
- Confirm you are **not** providing MCLK from the ESP32
- Check that the XMOS firmware is flashed correctly (DSP Status = OK)

### Wi-Fi connection issues

- Use **2.4 GHz** Wi-Fi (ESP32-S3 does not support 5 GHz)
- Set AP security to **WPA2-PSK (AES)** — avoid WPA3 or mixed mode
- Disable **802.11r/k/v** (fast roaming) on the AP
- Try a different Wi-Fi channel (1 or 6)
- An external antenna on the XIAO ESP32-S3 significantly improves stability

### Playback stuttering or delayed start

- This is usually a Wi-Fi bandwidth/latency issue
- Ensure the device has a strong Wi-Fi signal (RSSI > -60 dBm)
- The media pipeline buffers audio before playback; a 1–2 second startup delay is normal

## License

This project is based on work by [formatBCE](https://github.com/formatBCE).
See the upstream repositories for their respective licenses.

## Credits

- [formatBCE](https://github.com/formatBCE) — voice satellite core, XVF3800 I2C component, and the original integration architecture
- [Seeed Studio](https://www.seeedstudio.com/) — reSpeaker Flex hardware and XMOS firmware
- [linkedupbits](https://github.com/linkedupbits) — Flex I2S clock measurements and working 16 kHz configuration ([issue #2 comment](https://github.com/respeaker/reSpeaker_Flex/issues/2#issuecomment-6457437208))
