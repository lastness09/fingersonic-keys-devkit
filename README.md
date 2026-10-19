# Fingersonic Keys Developer Guide

Public hardware and firmware-development documentation for Fingersonic Keys. **Documentation only: this repository does not distribute Fingersonic product firmware source, bootloader source, system images, encryption keys or production tools.** Build your own application using the documented interfaces and public libraries.

[Developer Kits](https://fingersonic.com/developers/) · [Guitar developer kit](https://github.com/lastness09/fingersonic-guitar-devkit) · [Official Keys updater](https://update.fingersonic.cn/FingersonicUploader/)

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
7. Enter App1 using the gesture appropriate for the production date. Use the official Keys updater's documented custom-firmware workflow where supported; verify the displayed device and upload type before sending a file. This guide does not establish a raw custom-App0 HTTP endpoint or promise that every updater version accepts custom firmware.

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

## 中文：烧录与风险说明

**2027 年 1 月 1 日及之后生产、安装新版恢复引导程序的设备：**关机后按住右边的旋钮 **Encoder2**，再开机，持续按住至少 **0.5 秒**，可强制进入 App1 烧录模式，即使 App0 无法运行。此方式仍要求 bootloader、分区表和 App1 完好，不是芯片 ROM 下载模式。

**2027 年 1 月 1 日之前生产的设备：**必须由正常运行的 App0 调用 App1。官方固件的操作是：**按住左边第一个琴键 Key0，快速连续按左边旋钮 Encoder1 五次**。如果你的自定义固件无法启动，设备可能变砖，无法通过正常用户操作恢复，风险由你自行承担。建议开发之初就实现并测试自己的 App1 切换方式，例如开机早期识别按键操作、或在程序能够正常运行时通过蓝牙调用。蓝牙调用无法挽救完全不运行的 App0。保留可恢复路径，再进行后续开发。

本仓库仅公开 GPIO、搭建方法、分区参考与恢复说明，不公开官方产品固件源码、恢复引导源码、量产脚本、密钥或服务器凭据。

## Support

[Fingersonic](https://fingersonic.net/) · [Developer Kits](https://fingersonic.com/developers/) · [support@fingersonic.net](mailto:support@fingersonic.net)
