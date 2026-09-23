# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single ESPHome configuration (`m5stickc3plus.yaml`) that turns an M5Stack **StickS3** (ESP32-S3; despite the filename, not a StickC Plus) into a Home Assistant Assist voice satellite, with a push-to-talk button, the onboard mic and speaker, and a status display on the LCD. There is no application code or test suite. `README.md` is the user-facing documentation, so keep it in sync when behaviour, pins or tuning values change.

Developed against **ESPHome 2026.9.0** using the ESP-IDF framework.

## Commands

Wi-Fi and API credentials come from `secrets.yaml` (not committed), which must define `wifi_ssid`, `wifi_password` and `ha_api_pw`.

```bash
esphome config m5stickc3plus.yaml    # validate (closest thing to a lint/test)
esphome compile m5stickc3plus.yaml   # build; first build fetches Roboto from Google Fonts
esphome run m5stickc3plus.yaml       # flash (USB first time, OTA after)
esphome logs m5stickc3plus.yaml      # view device logs
```

Hardware behaviour can only be verified on a device. When checking that a change works, look for the log lines listed in the README (I2C scan finds `0x18`, `0x68` and `0x6E`; PSRAM shows 8192 KB; the voice assistant goes through TTS stream start/end and back to `IDLE`).

## Architecture: boot ordering is the critical part

The StickS3's ES8311 codec is powered through the **M5PM1 PMIC (0x6E)**, so the codec isn't on the I2C bus until the PMIC turns on its power rail. `esphome.on_boot` uses priorities to run this setup in the right order:

1. **priority 850** (after I2C at 1000, before ES8311 setup at ~600): a lambda writes raw M5PM1 registers through the `m5pm1` `i2c_device`. It disables PMIC I2C sleep and the watchdog, then sets PMIC GPIO2 high to turn on the L3B rail that powers the ES8311. It uses the `es8311_probe` `i2c_device` to check that the codec now responds. It also sets up PMIC GPIO3 (AW8737 speaker amp) as an output but leaves it **off**, then sets the `pm1_amp_ready` global. Reads and writes are retried because the M5PM1 can NAK the first transaction after sleeping.
2. **priority 500** (after the codec is set up): turns the amp on (reg 0x11, bit 0x08), but only if `pm1_amp_ready` is set.
3. **priority -100**: sets the DAC volume to 80%.

If you move, reorder or change these priorities, the ES8311 is likely to fail with "Failed to initialize".

## Other non-obvious constraints

- **Two I2S buses** (`i2s_input` and `i2s_output`) share the LRCLK, BCLK and MCLK pins via `allow_other_uses`. A single shared bus causes "Parent bus is busy".
- Keep `use_microphone: false` on the ES8311. The mic uses the analog input, and `true` switches ESPHome to the PDM path.
- `voice_assistant.speaker` must be set, otherwise TTS comes back as a URL and doesn't play on the device.
- Speaker `buffer_duration: 1s` and `timeout: 3s` stop long TTS replies being cut off. Smaller values truncated them.
- **Push-to-talk**: pressing the button starts a session with `silence_detection: true`, and VAD decides when speech has ended. Don't call `voice_assistant.stop` on release: in 2026.9.0 that stops the whole pipeline instead of submitting what was heard. On press, any session that is still running is stopped first.
- The `va_listen_timeout` script (8 s, `mode: restart`) recovers sessions stuck in `STREAMING_MICROPHONE`. The `on_stt_end`, `on_error` and `on_idle` triggers stop it.
- **Display**: `update_interval: never`. The `va_screen_state` global (0 Ready, 1 Listening, 2 Heard, 3 Thinking, 4 OK, 5 Failed) is set in the voice assistant triggers, each followed by `component.update: sticks3_display`. The display lambda switches on that value. `last_heard` and `last_error` hold the text shown on screen. Any new state needs a matching `case` in the lambda.
