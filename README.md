# Fingersonic Keys Developer Guide

Public hardware and firmware-development documentation for Fingersonic Keys. **Documentation only: this repository does not distribute Fingersonic product firmware source, bootloader source, system images, encryption keys or production tools.** Build your own application using the documented interfaces and public libraries.

[Developer Kits](https://fingersonic.com/developers/) · [Neon Orbit open-source example](#open-source-example-neon-orbit) · [Guitar developer kit](https://github.com/lastness09/fingersonic-guitar-devkit) · [Official Keys updater](https://update.fingersonic.cn/FingersonicUploader/)

## Start here: give this repository to your AI

Read the documents in this order. No private Fingersonic project is required.

| Document | What it supplies |
| --- | --- |
| This README | Scope, production-date recovery rules, GPIO overview and App1 upload interface |
| [Build from scratch](BUILD_FROM_SCRATCH.md) | Pinned ESP-IDF baseline, complete small-project file contents, recovery scaffold, build commands and App0 artifact checks |
| [Hardware contract](HARDWARE_CONTRACT.md) | Complete matrix contact/key table, scan ownership rules, event model, encoders, LED/audio timing, MIDI and storage constraints |
| [Partition reference](partitions.csv) | Exact local build partition layout; never upload it to the device |
| [Validation and handoff](VALIDATION.md) | Compilation evidence, hardware acceptance tests and required AI deliverables |

The kit supports independently written controllers, instruments and other App0 applications. It is not a promise to reproduce every feature, sound or calibration of the official firmware. Start with the documented no-external-dependency scaffold, then add the features your application needs. **Synthesis is open to your own design; sample-based instruments must use SF2 sound banks because the installed App1 uploader supports only `.bin` firmware and `.sf2` banks.**

### Choose what to build

| Direction | Possible projects | Resource requirement |
| --- | --- | --- |
| Algorithmic synthesis | FM, subtractive, additive, generated-wavetable or physical-model instruments | Your independently implemented App0 generates audio; no sample bank is required |
| Sample-based instrument | Piano, drums, multisampled instruments or sound-effects keyboard | Implement SF2 playback and package uploaded sample banks as `.sf2` |
| MIDI and performance | USB/BLE controller, chord controller, arpeggiator, step sequencer or generative performance tool | Implement the selected MIDI/performance behavior in App0; respect the same upload and storage constraints |

For a sample-based project, convert WAV/AIFF or other source recordings into an SF2 bank on the computer before uploading. Direct WAV, AIFF, MP3, ZIP or arbitrary asset uploads are not supported by App1. A `.bin` upload is executable App0 firmware, not a general resource container. Do not work around the format rule by renaming assets or assuming a generic file-transfer service exists. No microphone or audio-input interface is documented, so live audio recording is not an assumed capability.

## Open-source example: Neon Orbit

[Neon Orbit - Arpeggiator & FM Synth](https://github.com/lastness09/fingersonic-keys-neon-orbit) is a complete, MIT-licensed community firmware for Fingersonic Keys. It demonstrates ten arpeggiator patterns, ten FM sounds, velocity-sensitive playing, encoder controls, note-reactive LEDs and a dedicated App1 recovery gesture. The maintainer has confirmed testing on the instrument; its publication build and 28 host tests also pass.

Use this guide for the hardware interfaces and recovery requirements, then use the example repository for its complete source, controls, tests and reproducible build. Neon Orbit pins **Arduino ESP32 core 3.3.2** and **Adafruit NeoPixel 1.15.1**; that is a separately verified toolchain from this guide's ESP-IDF baseline. It generates audio directly and does not require an SF2 bank or PSRAM.

[Browse the example source](https://github.com/lastness09/fingersonic-keys-neon-orbit) · [Controls and build instructions](https://github.com/lastness09/fingersonic-keys-neon-orbit#readme) · [Download firmware](https://github.com/lastness09/fingersonic-keys-neon-orbit/releases/tag/v0.1.0) · [Community firmware page](https://fingersonic.com/firmware/3/) · [Watch the hardware demo](https://www.youtube.com/shorts/pRCk6zMpI8U)

For AI-assisted development, provide both this guide and the Neon Orbit repository. Ask the AI to retain the documented GPIO/partition contract and recovery path while changing the musical behavior. Review the example's MIT license and the licenses of any dependencies you reuse. The example is published separately; the official Fingersonic product firmware remains private.

## Develop with an AI coding assistant

Give your AI coding assistant this repository URL and describe the instrument, controller or application you want to build. This guide is the hardware contract for a **new, independently implemented App0 firmware project**. It does not require access to Fingersonic's private product firmware. Hardware interface documentation does not grant access to, or publish, the official firmware source.

You can start with this prompt:

> Read https://github.com/lastness09/fingersonic-keys-devkit and follow its links to BUILD_FROM_SCRATCH.md, HARDWARE_CONTRACT.md, partitions.csv and VALIDATION.md before writing code. If you cannot open a linked document, ask me to provide it rather than guessing. Develop a new Fingersonic Keys App0 application that implements the behavior I describe below. Use the documented GPIO mapping, 24-key dual-contact matrix, two encoders, 24 LEDs, I2S output and USB/BLE MIDI interfaces as needed. Use public libraries and original application code. Begin with the pinned ESP-IDF v5.3.1 baseline unless my requirements justify a separately verified toolchain. Produce a complete project with pinned dependencies, build instructions and an App0 BIN that fits the 2 MiB slot. Algorithmic synthesizers can use an independently designed sound engine. If my project uses samples, implement SF2 playback and use SF2 banks: App1 accepts only executable `.bin` firmware and `.sf2` sound banks, not WAV/MP3 files or arbitrary assets. Preserve the factory bootloader, partition table, existing sound bank and App1 uploader. Do not use alternating-slot OTA that overwrites App1 or format storage on mount failure. Implement and test an App1 recovery gesture before optional features. Power, charging and battery-level management are handled by a separate chip and require no custom application management. Do not invent battery GPIOs or assume unavailable board features. State assumptions, identify any missing information and provide a hardware test checklist. Report which checks you actually ran and distinguish compilation from hardware verification. My desired behavior is: [describe your application here].

### Requirements for AI-generated projects

- Deliver a self-contained project: project files, application entry point, build configuration, exact dependency versions and commands to build from a clean checkout. The proprietary Fingersonic application is not a dependency.
- Treat the GPIO and partition tables below as fixed interfaces. Use a 16 MB flash target. Prove recovery with the internal-RAM scaffold first; enable OPI PSRAM when your application needs it, with a runtime capacity check and an internal-RAM fallback. Keep the compiled App0 image within `0x200000` bytes.
- Initialize the recovery gesture before optional peripherals, sound loading or other work that could fail. Check the OTA partition selection result before restarting into App1. Do not make normal recovery depend solely on Bluetooth.
- Bring up one interface at a time: recovery, encoders, matrix note detection, LEDs, MIDI and audio. Add velocity and higher-level musical behavior after basic input and output are working.
- Use public Arduino or ESP-IDF interfaces for peripheral access. Library examples may need adaptation to the selected core version; do not assume that all upstream releases work together.
- Keep the two contacts of each key distinct until velocity calculation is complete. Apply the documented key-index swaps. Match LED ordering against the physical device during testing.
- Produce an application BIN for the existing App0 slot, not a merged full-flash image. Do not include factory provisioning, erase-all commands or eFuse changes in the normal installation workflow.
- Document your recovery entry gesture, installation steps, tested hardware production date and dependency versions. If a required uploader detail is unverified, identify it rather than inventing an endpoint or flashing command.

### Power, charging and battery level

**Power, charging and battery-level management are handled by a separate chip. Custom App0 firmware does not need to implement or manage them.** Do not add charger control, battery voltage measurement, fuel-gauge drivers or a software power-management task as a prerequisite for using this hardware. No battery ADC pin, charge-control GPIO or battery-chip bus protocol is specified by this guide; do not assign spare-looking pins to those functions.

## Read this before installing custom firmware

| Device production date | Entering the preinstalled App1 uploader |
| --- | --- |
| On or after January 1, 2027 | With power off, hold the **right-hand knob (Encoder2)**, power on, and keep holding for at least **0.5 seconds**. The recovery bootloader can enter App1 **even if App0 cannot run**. |
| Before January 1, 2027 | Entry depends on **running App0 firmware**. With official firmware running, hold the **leftmost keyboard key (Key0)** and quickly press the **left-hand knob (Encoder1) five times**. |

The January 2027 distinction is the manufacturer's production policy. The newer method requires the recovery bootloader to be installed; the date printed on a document or the date you purchased the unit does not upgrade an older bootloader.

**Older-device risk:** if your custom App0 firmware cannot start and has no working way to switch to App1, you may brick the device and lose the normal user recovery path. You accept this risk when installing custom firmware. Before experimenting, implement and test your own App1 entry mechanism, such as a button gesture early in startup or a Bluetooth command. A Bluetooth command only helps while your firmware and Bluetooth stack still run; it cannot recover a completely non-running App0. Preserve a known-good firmware image and test switching back before adding complex features.

The newer startup gesture selects **App1 (ota_1)** for that boot. It is **not ESP32-S3 ROM download mode**. It still requires a valid bootloader, partition table and App1. A missing or damaged App1 is not repaired by holding Encoder2. Do not overwrite App1, the bootloader, partition table or device security settings.

## Hardware interfaces / GPIO

Target: **ESP32-S3, 16 MB flash, OPI PSRAM**. GPIO numbers below are MCU GPIO numbers, not connector pin positions. Digital interfaces use the ESP32-S3 3.3 V logic domain; do not apply 5 V directly to GPIO. Confirm board wiring before attaching external hardware.

| Interface | Signal | GPIO | Notes |
| --- | --- | --- | --- |
| I2S audio output | BCLK | 40 | Bit clock |
| I2S audio output | DATA OUT | 41 | Connects to DAC data input |
| I2S audio output | LRCLK / WCLK | 42 | Word clock |
| I2S audio input | DIN | Not assigned | No configured audio input |
| Addressable LEDs | DATA | 2 | 24 LEDs, GRB, 800 kHz |
| Encoder1 (left) | CLK / A | 5 | Input with pull-up |
| Encoder1 (left) | DT / B | 16 | Input with pull-up |
| Encoder1 (left) | Switch | 13 | Active LOW, pull-up |
| Encoder2 (right) | CLK / A | 1 | Input with pull-up |
| Encoder2 (right) | DT / B | 3 | Input with pull-up; retain correct startup state |
| Encoder2 (right) | Switch | 38 | Active LOW; newer bootloader recovery input |
| Keyboard scan | Drive columns 0–5 | 12, 11, 10, 8, 7, 6 | Order matters |
| Keyboard scan | Sense inputs 0–7 | 48, 47, 39, 21, 18, 17, 15, 14 | Paired contacts; order matters |

Encoder common terminals connect to GND. The 6-column / 8-input matrix provides **24 velocity-sensitive keys**, using two contacts per key rather than 48 independent keys.

### Keyboard scanning

Use column order `[12, 11, 10, 8, 7, 6]` and input order `[48, 47, 39, 21, 18, 17, 15, 14]`. Pair adjacent sense inputs: `(48,47)`, `(39,21)`, `(18,17)`, `(15,14)`. Each pair covers six keys, one per column. Initial logical index = `floor(inputIndex / 2) * 6 + columnIndex`; contact stage = `inputIndex % 2`. The inspected board mapping swaps indices **5 ↔ 6** and **17 ↔ 18** after that calculation. Verify physical ordering on your hardware.

Drive one column HIGH and read the sense inputs with pull-downs. Release the column before scanning the next one. Avoid driving opposing levels through closed contacts. Debounce both contacts, timestamp their edges and derive velocity from the interval. Confirm contact order experimentally rather than assuming every key closes the same contact first. Start with note detection, then tune velocity and reject bounce.

### Audio, LEDs and MIDI

The inspected audio configuration uses **I2S master TX, Philips timing, 44,100 Hz, stereo, signed 16-bit PCM**, with no assigned MCLK or DIN pin. Start with a low-level test tone and verify I2S timing before adding synthesis. The LED chain uses 24 GRB pixels at 800 kHz on GPIO2; begin at low brightness and verify pixel order. See the [hardware contract](HARDWARE_CONTRACT.md) for buffer layout, key/LED ordering and unsupported peripheral assumptions.

Use ESP32-S3 native USB with **USB-OTG (TinyUSB)** for USB MIDI. The standard ESP32-S3 native USB signals are GPIO19 (D−) and GPIO20 (D+); confirm PCB routing before connecting external USB wiring. BLE MIDI can be implemented with a public BLE-MIDI library. Neither USB MIDI nor BLE MIDI automatically supplies an App1 recovery command: implement and test that in your own App0.

## Set up your own firmware project

This is a hardware guide, not a buildable copy of the proprietary application. Create a fresh project and bring up the documented interfaces incrementally.

1. For the reproducible starting point, follow [Build from scratch](BUILD_FROM_SCRATCH.md) using **ESP-IDF v5.3.1**. Arduino is an alternative requiring its own pinned core, library versions and build validation. Do not treat legacy project notes as a dependency lock.
2. Select an **ESP32-S3** target, **16 MB flash** and the custom partition layout matching the factory device. When enabling PSRAM, select **OPI / Octal Mode**. Check the actual PSRAM capacity on your unit; this guide does not establish its size.
3. For Arduino USB MIDI, select **USB Mode: USB-OTG (TinyUSB)**. Set **USB CDC On Boot**, **USB Firmware MSC On Boot** and **USB DFU On Boot** to **Disabled** for the recorded product configuration. Plan your diagnostic output accordingly.
4. Choose only the public libraries your application needs. The ESP-IDF baseline requires no external libraries. For an Arduino application, options include [Adafruit NeoPixel](https://github.com/adafruit/Adafruit_NeoPixel), [Control Surface](https://github.com/tttapa/Control-Surface), [Arduino BLE-MIDI](https://github.com/lathoub/Arduino-BLE-MIDI), and the [Arduino MIDI Library](https://github.com/FortySevenEffects/arduino_midi_library). Public upstream libraries retain their respective licenses. Optional MIDI, synthesis and filesystem libraries are not covered by the baseline compilation result; pin and verify their versions separately.
5. Implement and test **App1 switching first**, then encoders, keyboard scanning, LEDs, USB/BLE MIDI and I2S audio. Keep lengthy work out of audio callbacks and avoid excessive logging during audio playback.
6. Compile **App0 only**. Its allocated slot is **2 MiB (0x200000 bytes)**; check the built application's size before attempting installation. Do not upload a merged flash image or an ELF file as App0.
7. Enter App1 using the gesture appropriate for the production date. Upload your application BIN through the preinstalled App1 uploader as described below. The official web updater provides official recovery files; it does not supply your custom application's source or build it for you.

### Upload your custom App0 BIN through App1

The current official Keys updater sends files to the preinstalled device uploader using this interface:

| Item | Value |
| --- | --- |
| Device hotspot | `Fingersonic` in the current official updater instructions; older firmware notes also use `FingersonicUploader` |
| Hotspot connection instructions | Follow the connection instructions shown by the official Keys updater |
| Device address | `http://192.168.4.1` |
| Upload request | `POST /upload`, `multipart/form-data` |
| File field | `file` |
| Application file | An App0 application image with a `.bin` filename, no larger than `0x200000` bytes |
| Sound-bank file | `.sf2`; separate from the application BIN |

These are the only supported upload file types. For sample-based applications, keep SF2 as the bank format and implement a compatible SF2 reader/player in your own App0. Do not assume that a different engine makes new upload extensions available.

Prepare your BIN before switching away from your normal network. Enter App1, connect the computer to the device hotspot, then upload one file. For example, in PowerShell on Windows:

```powershell
curl.exe --fail --show-error -F "file=@C:\path\to\your-app.bin" http://192.168.4.1/upload
```

On macOS or Linux, use `curl` with the same arguments and your local file path. Let the upload finish and allow the device to restart. Do not send repeated uploads while it is processing. The current official web client uses the filename extension to distinguish BIN from SF2. This interface is verified against that client's request format; custom firmware installation has not been tested on physical hardware as part of this documentation release. Check the response and test your device's actual App1 version. A browser `no-cors` request does not by itself prove successful flashing.

Do not upload bootloader, partition-table, filesystem or merged full-flash BINs through the App0 workflow. To return to the official application, enter App1 again and use the official Keys updater.

### Factory partition layout (reference)

| Label | Type / subtype | Offset | Size | Flags |
| --- | --- | --- | --- | --- |
| nvs | data / nvs | 0x9000 | 0x5000 | |
| otadata | data / ota | 0xE000 | 0x2000 | |
| app0 | app / ota_0 | 0x10000 | 0x200000 | encrypted |
| app1 | app / ota_1 | 0x210000 | 0x100000 | encrypted |
| spiffs | data / spiffs | 0x310000 | 0xBF0000 | encrypted |
| coredump | data / coredump | 0xF00000 | 0x10000 | |

The filesystem partition label is `spiffs`, while the inspected application mounts **LittleFS**. A partition label alone does not establish the filesystem format. Use these values to configure and check your build; **do not reflash the factory partition table**. Existing devices may use flash encryption; ordinary plaintext serial flashing instructions are not interchangeable with the factory provisioning process. Never change eFuses as part of normal custom App0 development.

**App1 is not a spare OTA slot.** Do not apply a generic alternating-slot OTA example from App0: it can select App1 as the next write target and overwrite the uploader. Use App1's upload interface to replace App0. Do not automatically format LittleFS or erase the entire NVS partition when initialization fails. Preserve the owner's data and offer a recovery path.

### App1 recovery in your own application

Using the public ESP-IDF OTA API, locate the application partition with subtype `ESP_PARTITION_SUBTYPE_APP_OTA_1`, verify it exists, call `esp_ota_set_boot_partition`, check that the result is `ESP_OK`, then restart. Only restart into App1 after successful selection. This describes public APIs without distributing product implementation code.

For older units, put a physical recovery gesture near the beginning of startup, before sound-bank loading and optional peripherals. Optionally add a Bluetooth-triggered entry as a convenience. Test with a deliberately unavailable sound bank and with optional peripherals disconnected. These measures reduce risk but do not provide bootloader-independent recovery on older devices.

## Sound banks and returning to official firmware

The installed App1 uploader accepts only `.bin` application firmware and `.sf2` sound banks. **If you build a sample-based instrument, its uploaded banks must be SF2 and your App0 must implement SF2 playback.** Convert source recordings into SF2 on the host computer. Direct uploads of WAV, AIFF, MP3 or other resource formats are unavailable.

An algorithmic synthesizer can implement its own oscillators, filters, envelopes and modulation without needing a sample bank. Choosing a different synthesis engine does not add file formats to the unchanged App1 uploader. Compatibility with official sounds requires your own validation.

The official Keys sound-bank convention is one SF2 file and Bank 0, with a conservative **7 MB maximum** in the recorded usage notes. This is not a universal SF2-format limit. Filesystem partition capacity does not mean every file that fits can be loaded into PSRAM; validate memory use and the actual uploader version. A custom SF2 implementation must state which presets, sample encodings, generators and modulators it supports rather than promise every SF2 file will work.

Use the [official Keys updater](https://update.fingersonic.cn/FingersonicUploader/) to restore official App0 firmware. Restore the official sound bank separately if you replaced it. The normal recovery path requires reaching App1. On older devices with non-running custom App0, the official button gesture cannot execute and normal user recovery may be unavailable.

## Recovery checklist

- **Devices produced on or after January 1, 2027 with the recovery bootloader:** hold the right-hand knob, **Encoder2**, while powering on, and keep holding for at least **0.5 seconds**. This enters App1 even if App0 cannot run, provided the bootloader, partition table and App1 are intact.
- **Devices produced before January 1, 2027:** App0 must run to select App1. With official firmware, hold **Key0**, the leftmost keyboard key, and quickly press **Encoder1**, the left-hand knob, **five times**. Custom firmware that cannot start may leave the device bricked without normal user recovery. You accept this risk. Implement and test an early-startup button gesture or another App1 entry mechanism before experimenting. A Bluetooth command only works while your firmware and Bluetooth stack are running.
- This repository publishes GPIO mappings, setup instructions, partition references and recovery guidance only. It does not publish official product firmware source, recovery bootloader source, production scripts, keys or server credentials.

## Support

[Fingersonic](https://fingersonic.net/) · [Developer Kits](https://fingersonic.com/developers/) · [support@fingersonic.net](mailto:support@fingersonic.net)
