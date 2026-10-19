# Fingersonic Keys hardware contract for independent firmware

Use this with the [README GPIO table](README.md#hardware-interfaces--gpio). Values here describe the supported application interfaces, not the internal design of the official instrument. Implement your own drivers and musical behavior.

## Board boundaries

- ESP32-S3, 16 MB flash and OPI PSRAM. Detect available PSRAM capacity at runtime. Flash capacity is not PSRAM capacity.
- 24 dual-contact keys, two quadrature encoders with switches, 24 serial LEDs and a three-wire I2S audio output.
- Power, charging and battery-level management are provided by an independent chip. App0 does not need a battery driver, charger driver or power latch. No battery-control pin or battery telemetry protocol is assigned to the application here.
- A display, SD card, microphone, I2S input, documented external UART MIDI connector and arbitrary expansion header are not part of this interface contract. Do not import Guitar's touch controller, joystick, display or pin mapping into Keys firmware.
- Use only the assigned pins. GPIO19/20 are the ESP32-S3 native USB signals. Internal flash/PSRAM signals, strapping pins and pins not listed as usable must not be repurposed merely because they appear free in a generic ESP32-S3 board example.

## Keyboard contact map

There are six driven columns and four pairs of sense inputs. During a scan, an asserted contact reads HIGH when its selected column is HIGH; inactive sense lines use pull-downs. Index contacts and columns from zero.

| Sense pair | Contact A GPIO | Contact B GPIO | Key indices for column GPIOs 12, 11, 10, 8, 7, 6 |
| --- | --- | --- | --- |
| 0 | 48 | 47 | 0, 1, 2, 3, 4, 6 |
| 1 | 39 | 21 | 5, 7, 8, 9, 10, 11 |
| 2 | 18 | 17 | 12, 13, 14, 15, 16, 18 |
| 3 | 15 | 14 | 17, 19, 20, 21, 22, 23 |

The table already includes the key-index swaps 5/6 and 17/18. Do not swap them a second time. Equivalently, calculate `pairIndex * 6 + columnIndex` and then apply each swap once. Logical key indices increase from the leftmost key. A reference chromatic mapping is MIDI notes **48 through 71**: `note = 48 + keyIndex`, on MIDI channel 1. Musical mappings, transposition and octave policy are application choices.

### Scanning sequence

1. Begin with all six column pins and all eight sense pins as inputs with pull-downs. Ensure no column remains HIGH from an earlier scan.
2. Select one column, make only that column an output HIGH, allow the wiring to settle and read all eight sense inputs. Other columns remain inputs with pull-downs.
3. Release the selected column to an input with pull-down and allow signals to settle again. Read the sense lines a second time. Accept a contact for the selected column only if it was HIGH while driven and is LOW after release. This ownership check helps reject capacitive coupling and phantom contacts.
4. Complete the six-column pass, retaining a separate state for each of the two contacts of all 24 keys. Keep all unselected columns released. Do not leave one output HIGH while actively pulling connected sense lines LOW.
5. Debounce and timestamp accepted edges with a monotonic microsecond timer. Use a wrap-safe time difference. Keep the scan period short relative to the fastest contact interval you intend to measure. Measure and record a complete scan's duration on hardware before claiming velocity accuracy.

Use a conservative settling interval for the first test, inspect raw contact states and shorten it only after checking stability. Exact settling, debounce and velocity thresholds depend on the board and are calibration parameters, not fixed pinout facts. A slow task that polls each key every 10 ms is unsuitable for accurate dual-contact velocity.

### Independent per-key event model

- Idle: neither contact is asserted; no note is active.
- First accepted contact: store which contact arrived and its timestamp. Either member of the pair can be first; do not hard-code one ordering without measuring it.
- Second accepted contact while the first is still held: calculate the interval and emit exactly one Note On. Smaller intervals should map to greater velocity. Clamp MIDI Note On velocity to **1-127**. Velocity zero is Note Off, not a quiet note.
- Only one contact held beyond a configurable timeout: define an explicit policy. For example, emit one low-velocity event to support very slow presses, or reject the incomplete press. State this choice in your project; it is not a universal hardware guarantee.
- Release: after both contacts have stably returned LOW, emit one Note Off if a note was active, then reset the timestamp and contact state. One contact releasing earlier must not create a duplicate note.
- Reconnect, mode changes and error recovery: clear active notes and send the relevant All Notes Off / All Sound Off messages so interrupted playback does not leave stuck notes.

Write your own velocity curve. This repository does not publish the official velocity calibration, sound engine, chord engine or performance algorithms. All 24 keys require a slow-press, fast-press, release and multi-key test.

## Encoders

| Encoder | A / CLK | B / DT | Switch |
| --- | --- | --- | --- |
| Left / Encoder1 | GPIO5 | GPIO16 | GPIO13 |
| Right / Encoder2 | GPIO1 | GPIO3 | GPIO38 |

Enable pull-ups on A, B and switch inputs. Common terminals connect to ground; switches are pressed LOW. Use a quadrature transition decoder and reject invalid state jumps. Debounce switches separately from rotation. The number of electrical transitions per mechanical detent and rotation sign should be measured on your unit; do not assume a single edge means one detent. Validate direction on both knobs before assigning musical actions.

For a custom recovery gesture, a deliberate hold of both switches can be implemented without initializing the key matrix, Bluetooth, audio or storage. [The build scaffold](BUILD_FROM_SCRATCH.md) demonstrates this with public OTA APIs. It remains application-dependent on older units.

## LED output

The chain uses GPIO2, 24 pixels, **GRB byte order**, **800 kHz**, 8 bits per color. Use an established NeoPixel-compatible driver or ESP-IDF RMT implementation rather than bit-banging during time-critical scanning or audio.

Logical key index and LED index use the same 0-23 order in the inspected interface. Begin with a one-pixel walking test to confirm physical ordering and avoid applying the matrix swaps again to LED indices. Start at low brightness. Driver timing and maximum global brightness are application choices; the board's maximum safe LED current is not specified here. No separate LED-power-enable pin is documented.

## Audio output

Use **I2S master TX**, standard **Philips I2S timing** (one-bit delay), **44,100 frames per second**, two channels and signed **16-bit PCM** per channel. BCLK is GPIO40, WS/LRCLK is GPIO42, and serial data output is GPIO41. Polarity is not inverted. No MCLK GPIO and no audio input are assigned. With 16-bit stereo slots, BCLK is 1.4112 MHz and WS is 44.1 kHz.

Use ESP-IDF 5.3.1's `driver/i2s_std.h`, `I2S_ROLE_MASTER`, `I2S_STD_PHILIPS_SLOT_DEFAULT_CONFIG(I2S_DATA_BIT_WIDTH_16BIT, I2S_SLOT_MODE_STEREO)` and unused MCLK/DIN pins. Do not substitute MSB-justified mode without checking the DAC. Buffers contain alternating left/right 16-bit samples. For the first audible test, duplicate a low-level waveform into both channels, starting and stopping with silence to limit clicks. Never cast floating-point sample memory to an integer PCM buffer.

Choose enough DMA buffering to keep output continuous and write audio from a task that can meet the buffer deadline. Keep DMA descriptors and required DMA-accessible buffers in internal memory. Place large samples in PSRAM only after verifying it is available. Do not run filesystem access, logging, LED updates or BLE work in an audio interrupt. A sample player or synthesizer is your application, not a required private component.

The exact DAC part, downstream amplifier, speaker/headphone routing and optional mute controls are not established by this documentation. No I2C codec setup, amplifier-enable pin or analog volume-control GPIO is specified. Use the documented digital I2S interface; do not invent extra control signals. Confirm left/right behavior and output level on hardware.

### Synthesis versus sample playback

You may implement an original FM, subtractive, additive, generated-wavetable or physical-model synthesizer, or another algorithmic engine that generates PCM for I2S. The keyboard, encoders and LEDs can provide your own performance interface.

For a sample-based instrument, **use SF2 sound banks**. The unchanged App1 uploader accepts only `.bin` executable App0 firmware and `.sf2` banks. It does not upload arbitrary WAV/AIFF/MP3 files, impulse-response files, sample folders or ZIP packs. Convert recordings into SF2 on the host and implement an SF2 reader/player in App0. Do not disguise an asset file as firmware by renaming it `.bin`.

An SF2 implementation needs sample-data loading, bank/program selection, note/velocity zones, tuning, envelopes and loop handling for the subset it claims to support. Choose a public SF2 engine or write your own; verify its license and memory usage. Document unsupported SF2 features and reject unsupported banks gracefully. A missing or incompatible bank must leave the recovery gesture working. No audio input or live-recording interface is documented.

## MIDI and optional app compatibility

USB MIDI uses the native USB device peripheral. Include an actual MIDI streaming interface and descriptors through TinyUSB or a public library; CDC alone is not MIDI. BLE MIDI uses the standard BLE MIDI service and its timestamped packet format, not a plain serial characteristic. Native ESP-IDF does not gain either protocol merely by enabling Bluetooth or USB in `menuconfig`.

For a first project, support Note On/Off on channel 1, velocity 1-127, CC7 volume and an explicit All Notes Off path. Test Note On with velocity zero as Note Off when receiving external MIDI. Program changes and pitch bend are optional application behavior. Choose development USB identifiers according to the SDK/library's documented example policy; do not claim a new production VID/PID allocation or reuse the official product's identity as your own.

The official Fingersonic phone app is optional. A standalone keyboard, sound generator or MIDI controller can use the interfaces above without it. This guide does not promise compatibility with every official app feature, sound bank or undocumented command. A Bluetooth-triggered recovery command is your own protocol unless explicitly specified otherwise.

## Storage and persistent state

App0 is `ota_0` at `0x10000`, 2 MiB; App1 is `ota_1` at `0x210000`, 1 MiB. App1 is a dedicated uploader, **not a spare slot for ordinary alternating OTA updates**. From App0, a generic `esp_ota_get_next_update_partition(NULL)` update could target App1 and destroy the user recovery path. Do not use that pattern here. Enter App1 and let its supported upload handler replace App0.

Do not erase `otadata`, the partition table, App1 or the entire device. Use public OTA APIs for boot selection. Leave security settings and eFuses alone; the device and installed uploader handle their existing flash-encryption state. App1 upload expects an application BIN, not a pre-encrypted image for another device.

The `spiffs`-labeled data partition contains LittleFS in the inspected application. The baseline does not mount it. If adding storage, use a compatible LittleFS component, the existing partition label and **format-on-failure disabled**. A failed mount must not erase the owner's sound bank. Read-only access or a memory-only first project avoids that dependency. Do not apply a SPIFFS formatter because the partition is named `spiffs`. For sampling, locate the uploaded `.sf2` in that filesystem rather than hard-coding an undocumented filename, and keep sample-bank replacement within App1's SF2 upload workflow.

If using NVS, use a distinct application namespace and explicit key versions. Avoid the common example pattern that erases the entire NVS partition after an initialization error; this may destroy other stored settings. Handle errors and offer application-scoped reset behavior.

## Information that requires hardware confirmation

Production-date recovery support is the manufacturer's policy stated in the README. Actual PSRAM capacity, encoder detent resolution, audio routing, electrical settling margins and the installed App1's response body must be checked on the unit being used. These are not reasons to request private firmware source. An AI should expose them as measured configuration or acceptance checks and must not claim they were tested merely because its code compiles.
