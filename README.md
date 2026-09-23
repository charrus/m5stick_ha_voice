# M5StickS3 ESPHome Voice Assistant

Turn an **M5Stack StickS3** into a compact **Home Assistant Assist** voice satellite using **ESPHome**.

This project uses the StickS3’s onboard microphone, speaker, ES8311 audio codec, PSRAM and hardware button. It also includes the StickS3-specific power-management setup required to bring the audio subsystem online reliably.

> Tested during development with **ESPHome 2026.9.0**.

## Features

- Home Assistant Assist voice input
- Onboard StickS3 microphone
- Onboard StickS3 speaker
- ES8311 codec support
- StickS3 M5PM1 audio power initialization
- 8 MB PSRAM enabled
- Hardware push/tap-to-talk button on GPIO11
- Voice activity detection
- Local TTS playback through the onboard speaker
- Speaker output tuned to avoid clipping
- Automatic recovery from stale voice-assistant sessions
- OTA updates

## Hardware

This project targets the **M5Stack StickS3** based on the ESP32-S3.

### Audio and control pins

| Function | GPIO |
|---|---:|
| I2C SDA | 47 |
| I2C SCL | 48 |
| I2S MCLK | 18 |
| I2S BCLK | 17 |
| I2S LRCLK / WS | 15 |
| ES8311 ADC → ESP32 microphone data | 16 |
| ESP32 → ES8311 speaker data | 14 |
| Main button / PTT | 11 |
| Secondary button | 12 |

### I2C devices

A healthy boot should normally show:

| Address | Device |
|---|---|
| `0x18` | ES8311 audio codec |
| `0x68` | BMI270 IMU |
| `0x6E` | M5PM1 power-management IC |

Example:

```text
Found device at address 0x18
Found device at address 0x68
Found device at address 0x6E
```

## Why StickS3 needs extra setup

The StickS3 is not just a generic ESP32-S3 + ES8311 board.

Its audio hardware is powered through the **M5PM1** power-management IC. The audio rail must be enabled before ESPHome initializes the ES8311.

Without this initialization, ESPHome typically reports:

```text
ES8311 Audio Codec:
  Failed to initialize!
```

and the I2C scan may not show `0x18`.

The configuration in this project initializes the M5PM1 early in the boot process, enables the StickS3 audio rail, and only then allows the ES8311 to initialize.

## ESPHome requirements

The configuration uses:

- ESP-IDF
- ESP32-S3
- PSRAM
- `i2c_device`
- `i2s_audio`
- `es8311`
- `voice_assistant`
- Home Assistant API

Recommended base configuration:

```yaml
esp32:
  variant: esp32s3
  flash_size: 8MB
  framework:
    type: esp-idf

psram:
  mode: octal
  speed: 80MHz
  ignore_not_found: false

network:
  tcp_send_buffer: 32kB
```

On a successful boot, ESPHome should report:

```text
PSRAM:
  Available: YES
  Size: 8192 KB
```

## I2C

The StickS3 internal bus is:

```yaml
i2c:
  id: internal_i2c
  sda: GPIO47
  scl: GPIO48
  frequency: 100kHz
  scan: true
```

100 kHz has proven reliable with the StickS3 M5PM1 and ES8311 during testing.

## ES8311 codec

The onboard microphone is connected to the ES8311 analog microphone input.

For that reason:

```yaml
use_microphone: false
```

is intentional. In ESPHome, enabling this option selects the ES8311 PDM microphone path instead.

Example:

```yaml
audio_dac:
  - platform: es8311
    id: es8311_dac
    address: 0x18
    i2c_id: internal_i2c

    use_mclk: true
    use_microphone: false

    sample_rate: 16000
    bits_per_sample: 16bit
```

## Microphone

```yaml
microphone:
  - platform: i2s_audio
    id: sticks3_microphone
    i2s_audio_id: i2s_input

    i2s_din_pin: GPIO16
    adc_type: external
    pdm: false

    i2s_mode: primary

    sample_rate: 16000
    bits_per_sample: 16bit
    channel: left

    mclk_multiple: 256
```

## Speaker

The onboard speaker uses GPIO14 through the ES8311 DAC.

```yaml
speaker:
  - platform: i2s_audio
    id: sticks3_speaker
    i2s_audio_id: i2s_output

    i2s_dout_pin: GPIO14
    dac_type: external
    audio_dac: es8311_dac

    i2s_mode: primary

    sample_rate: 16000
    bits_per_sample: 16bit
    channel: left

    buffer_duration: 500ms
    timeout: 1s
```

### Speaker volume

The default ES8311 output level was too aggressive on the StickS3 and produced audible clipping.

A DAC level of **65%** has produced clean playback during testing:

```yaml
- audio_dac.set_volume:
    id: es8311_dac
    volume: 65%
```

## Voice Assistant

The speaker must be explicitly connected to `voice_assistant`.

Without this:

```yaml
speaker: sticks3_speaker
```

Home Assistant may return a normal MP3 TTS URL, but ESPHome will not stream the response to the local speaker.

Example:

```yaml
voice_assistant:
  id: va

  microphone:
    microphone: sticks3_microphone
    gain_factor: 4

  speaker: sticks3_speaker

  noise_suppression_level: 2
  auto_gain: 31dBFS
```

A successful interaction should include log entries similar to:

```text
Speech recognised as: "What time is it?"
Response: "07:33"
TTS stream start
i2s_audio.speaker: Starting
...
TTS stream end
Speaker has finished outputting all audio
State changed from RESPONSE_FINISHED to IDLE
```

## Button control

GPIO11 is used as the main voice button.

ESPHome 2026.9.0 has an important behavior to be aware of: using `voice_assistant.stop` on button release can terminate the entire Assist pipeline instead of behaving as a clean “release to send” action.

For reliability, this project currently uses **button-to-start with Home Assistant VAD handling the end of speech**.

Example:

```yaml
binary_sensor:
  - platform: gpio
    id: push_to_talk
    name: "Push to Talk"

    pin:
      number: GPIO11
      inverted: true
      mode:
        input: true
        pullup: true

    filters:
      - delayed_on: 20ms
      - delayed_off: 20ms

    on_press:
      - logger.log: "PTT pressed - starting fresh session"
      - voice_assistant.start:
          silence_detection: true

    on_release:
      - logger.log: "PTT released"
```

In practice this behaves like:

1. Press the button.
2. Speak.
3. Release the button.
4. Stop speaking.
5. VAD detects the end of speech.
6. Home Assistant processes the request.
7. The StickS3 speaks the response.

## Session watchdog

A stale STT session can otherwise leave the Voice Assistant in `STREAMING_MICROPHONE` and cause the next button press to be ignored.

A simple timeout script can reset it:

```yaml
script:
  - id: va_listen_timeout
    mode: restart

    then:
      - delay: 8s

      - if:
          condition:
            voice_assistant.is_running:
          then:
            - logger.log: "Voice listening timeout - resetting"
            - voice_assistant.stop:
```

Stop that watchdog after successful recognition, TTS or return to idle.

## Home Assistant

You need:

- Home Assistant with Assist configured
- ESPHome integration
- A working STT engine
- A working TTS engine
- An Assist pipeline assigned to the ESPHome device

The device should appear in Home Assistant automatically after the ESPHome API connection is established.

## Secrets

Keep Wi-Fi and API credentials in `secrets.yaml`.

Example:

```yaml
wifi_ssid: "YOUR_WIFI"
wifi_password: "YOUR_PASSWORD"

api_encryption_key: "YOUR_API_KEY"
ota_password: "YOUR_OTA_PASSWORD"
```

Then reference them from the main configuration:

```yaml
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
```

Do not commit your real `secrets.yaml`.

Add it to `.gitignore`:

```gitignore
secrets.yaml
.esphome/
```

## Flashing

Validate:

```bash
esphome config m5stack.yaml
```

Compile:

```bash
esphome compile m5stack.yaml
```

Initial flash over USB:

```bash
esphome run m5stack.yaml
```

After the first installation, OTA can be used normally.

## Troubleshooting

### ES8311 fails to initialize

Symptom:

```text
es8311.audio_dac is marked FAILED
```

Check that:

- the M5PM1 is visible at `0x6E`
- the StickS3 audio power rail is enabled before ES8311 setup
- GPIO47/GPIO48 are used for SDA/SCL
- I2C is running at 100 kHz

### `0x18` missing from I2C scan

The ES8311 audio rail may not yet be powered.

Verify the M5PM1 initialization runs before the codec setup.

### `ESP_ERR_NO_MEM`

Enable the StickS3 PSRAM:

```yaml
psram:
  mode: octal
  speed: 80MHz
```

The StickS3 should report 8192 KB.

### `Parent bus is busy`

The microphone and speaker are competing for the same ESPHome I2S parent.

Use separate I2S instances for input and output.

### `Cannot receive audio, buffer is full`

Usually follows an I2S speaker startup failure.

Fix the I2S bus ownership problem first.

### TTS URL is MP3 and nothing plays

Check that the Voice Assistant has:

```yaml
speaker: sticks3_speaker
```

When direct speaker output is active, Home Assistant/ESPHome should use the local streaming path rather than just return a media URL.

### Speaker audio is static or corrupted

Check:

```yaml
sample_rate: 16000
bits_per_sample: 16bit
channel: left
```

Also make sure microphone and speaker I2S clocking are compatible.

### Speaker audio clips

Reduce the ES8311 DAC volume.

65% has worked well during development:

```yaml
- audio_dac.set_volume:
    id: es8311_dac
    volume: 65%
```

### Second button press does nothing

Check the log for a Voice Assistant state other than `IDLE`.

A successful response should eventually end with:

```text
Speaker has finished outputting all audio
State changed from RESPONSE_FINISHED to IDLE
```

If the session remains active, use the watchdog described above.

## Current limitations

### True hold-to-talk / release-to-send

The intended UX is:

> hold → speak → release → send

However, with ESPHome 2026.9.0, `voice_assistant.stop` can terminate the Assist pipeline rather than cleanly finishing the STT stream and waiting for a response.

The reliable workaround is currently:

> press → speak → VAD detects silence → response

A small ESPHome patch or future upstream change may make true release-to-send possible without the VAD workaround.

## Repository layout

Suggested layout:

```text
.
├── README.md
├── m5stack.yaml
├── secrets.example.yaml
└── .gitignore
```

## Status

Working:

- [x] StickS3 boots under ESPHome
- [x] M5PM1 detected
- [x] ES8311 detected
- [x] ES8311 powers up reliably
- [x] 8 MB PSRAM
- [x] Onboard microphone
- [x] Home Assistant STT
- [x] Assist intent processing
- [x] TTS streaming
- [x] Onboard speaker
- [x] Clean speaker output at 65%
- [x] Repeated Assist sessions
- [x] OTA updates

Still being refined:

- [ ] True release-to-send PTT behavior
- [ ] Wake-word mode
- [ ] Display/UI feedback
- [ ] Battery status
- [ ] Secondary button behavior

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

## Acknowledgements

Built around:

- ESPHome
- Home Assistant Assist
- M5Stack StickS3
- Espressif ESP32-S3
- Everest Semiconductor ES8311
