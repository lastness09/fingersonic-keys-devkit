# Fingersonic Keys Developer Guide

Public hardware and firmware-development documentation for Fingersonic Keys. **Documentation only: this repository does not distribute Fingersonic product firmware source, bootloader source, system images, encryption keys or production tools.** Build your own application using the documented interfaces and public libraries.

[Developer Kits](https://fingersonic.com/developers/) · [Guitar developer kit](https://github.com/lastness09/fingersonic-guitar-devkit) · [Official Keys updater](https://update.fingersonic.cn/FingersonicUploader/)

## Develop with an AI coding assistant

Give your AI coding assistant this repository URL and describe the instrument, controller or application you want to build. This guide is the hardware contract for a **new, independently implemented App0 firmware project**. It does not require access to Fingersonic's private product firmware. Hardware interface documentation does not grant access to, or publish, the official firmware source.

You can start with this prompt:

> Read https://github.com/lastness09/fingersonic-keys-devkit before writing code. Develop a new Fingersonic Keys App0 application that implements the behavior I describe below. Use the documented GPIO mapping, 24-key dual-contact matrix, two encoders, 24 LEDs, I2S output and USB/BLE MIDI interfaces as needed. Use public libraries and original application code. Produce a complete project with pinned dependencies, build instructions and an App0 BIN that fits the 2 MiB slot. Preserve the factory bootloader, partition table and App1 uploader. Implement and test an App1 recovery gesture before optional features. Power, charging and battery-level management are handled by a separate chip and require no custom application management. Do not invent battery GPIOs or assume unavailable board features. State assumptions, identify any missing information and provide a hardware test checklist. My desired behavior is: [describe your application here].

### Requirements for AI-generated projects

- Deliver a self-contained project: project files, application entry point, build configuration, exact dependency versions and commands to build from a clean checkout. The proprietary Fingersonic application is not a dependency.
- Treat the GPIO and partition tables below as fixed interfaces. Enable OPI PSRAM and use a 16 MB flash target. Keep the compiled App0 image within `0x200000` bytes.
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

The inspected audio configuration uses **44,100 Hz, stereo, 16-bit PCM**. Start with a low-level test tone and verify I2S timing before adding synthesis. The LED chain uses 24 GRB pixels at 800 kHz on GPIO2; begin at low brightness and verify pixel order.

Use ESP32-S3 native USB with **USB-OTG (TinyUSB)** for USB MIDI. The standard ESP32-S3 native USB signals are GPIO19 (D−) and GPIO20 (D+); confirm PCB routing before connecting external USB wiring. BLE MIDI can be implemented with a public BLE-MIDI library. Neither USB MIDI nor BLE MIDI automatically supplies an App1 recovery command: implement and test that in your own App0.

## Set up your own firmware project

This is a hardware guide, not a buildable copy of the proprietary application. Create a fresh project and bring up the documented interfaces incrementally.

1. Install Arduino IDE and the Espressif ESP32 board package. Internal project notes record **Arduino ESP32 Boards 2.0.18**; treat this as a legacy environment reference, not a guarantee that all modern library combinations work. Pin the actual core and library versions you validate. ESP-IDF is also an option, but requires your own project integration.
2. Select an **ESP32-S3** target, **16 MB flash**, **OPI PSRAM** and a custom partition layout matching the factory device. Check the actual PSRAM capacity on your unit; this guide does not establish its size.
3. For Arduino USB MIDI, select **USB Mode: USB-OTG (TinyUSB)**. Set **USB CDC On Boot**, **USB Firmware MSC On Boot** and **USB DFU On Boot** to **Disabled** for the recorded product configuration. Plan your diagnostic output accordingly.
4. Choose only the public libraries your application needs: [Adafruit NeoPixel](https://github.com/adafruit/Adafruit_NeoPixel), [Control Surface](https://github.com/tttapa/Control-Surface), [Arduino BLE-MIDI](https://github.com/lathoub/Arduino-BLE-MIDI), or the [Arduino MIDI Library](https://github.com/FortySevenEffects/arduino_midi_library). Public upstream libraries retain their respective licenses. There is no complete tested dependency lockfile in this documentation release.
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

Prepare your BIN before switching away from your normal network. Enter App1, connect the computer to the device hotspot, then upload one file. For example, in PowerShell on Windows:

```powershell
curl.exe --fail --show-error -F "file=@C:\path\to\your-app.bin" http://192.168.4.1/upload
```

On macOS or Linux, use `curl` with the same arguments and your local file path. Let the upload finish and allow the device to restart. Do not send repeated uploads while it is processing. The current official web client uses the filename extension to distinguish BIN from SF2. This interface is verified against that client's request format; custom firmware installation has not been tested on physical hardware as part of this documentation release. Check the response and test your device's actual App1 version. A browser `no-cors` request does not by itself prove successful flashing.

Do not upload bootloader, partition-table, filesystem or merged full-flash BINs through the App0 workflow. To return to the official application, enter App1 again and use the official Keys updater.

Legacy build notes mention a BLE-MIDI compatibility adjustment: convert the result of `characteristic->getValue()` through `.c_str()` into a `std::string` if a core/library type mismatch occurs. Apply this only to a dependency combination that actually needs it, then record the versions.

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

### App1 recovery in your own application

Using the public ESP-IDF OTA API, locate the application partition with subtype `ESP_PARTITION_SUBTYPE_APP_OTA_1`, verify it exists, call `esp_ota_set_boot_partition`, check that the result is `ESP_OK`, then restart. Only restart into App1 after successful selection. This describes public APIs without distributing product implementation code.

For older units, put a physical recovery gesture near the beginning of startup, before sound-bank loading and optional peripherals. Optionally add a Bluetooth-triggered entry as a convenience. Test with a deliberately unavailable sound bank and with optional peripherals disconnected. These measures reduce risk but do not provide bootloader-independent recovery on older devices.

## Sound banks and returning to official firmware

The official Keys application uses SF2 sound banks. Internal usage notes specify one SF2 file, Bank 0, with a conservative **7 MB maximum**. Filesystem partition capacity does not mean every file that fits can be loaded into PSRAM. A custom application may choose a different sound engine; compatibility with official sounds then requires your own validation.

Use the [official Keys updater](https://update.fingersonic.cn/FingersonicUploader/) to restore official App0 firmware. Restore the official sound bank separately if you replaced it. The normal recovery path requires reaching App1. On older devices with non-running custom App0, the official button gesture cannot execute and normal user recovery may be unavailable.

## Recovery checklist

- **Devices produced on or after January 1, 2027 with the recovery bootloader:** hold the right-hand knob, **Encoder2**, while powering on, and keep holding for at least **0.5 seconds**. This enters App1 even if App0 cannot run, provided the bootloader, partition table and App1 are intact.
- **Devices produced before January 1, 2027:** App0 must run to select App1. With official firmware, hold **Key0**, the leftmost keyboard key, and quickly press **Encoder1**, the left-hand knob, **five times**. Custom firmware that cannot start may leave the device bricked without normal user recovery. You accept this risk. Implement and test an early-startup button gesture or another App1 entry mechanism before experimenting. A Bluetooth command only works while your firmware and Bluetooth stack are running.
- This repository publishes GPIO mappings, setup instructions, partition references and recovery guidance only. It does not publish official product firmware source, recovery bootloader source, production scripts, keys or server credentials.

## Support

[Fingersonic](https://fingersonic.net/) · [Developer Kits](https://fingersonic.com/developers/) · [support@fingersonic.net](mailto:support@fingersonic.net)
