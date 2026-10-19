# Build an independent App0 from scratch

This document defines a reproducible ESP-IDF baseline without any Fingersonic product source. The short example below is newly written documentation code using public Espressif APIs. It is a recovery and build scaffold, not an implementation of the official instrument and not a playable firmware release.

## Pinned baseline

Use **ESP-IDF v5.3.1**, target **esp32s3**, and the tools installed by that release. Install from [Espressif's v5.3.1 getting-started guide](https://docs.espressif.com/projects/esp-idf/en/v5.3.1/esp32s3/get-started/index.html). Open that installation's activated ESP-IDF terminal. Run `idf.py --version`; it must report `ESP-IDF v5.3.1`. Other releases require a separate compatibility check.

No Arduino installation, external component download, private library, SoundFont, display driver or battery driver is required for this baseline. PSRAM is deliberately disabled for the first recovery test so application startup does not depend on external RAM initialization. Enable OPI PSRAM later using the settings below.

## Create these files

Use a new folder named `keys_custom`. Do not create it inside the official firmware project or import that project's files.

```text
keys_custom/
  CMakeLists.txt
  sdkconfig.defaults
  partitions.csv
  main/
    CMakeLists.txt
    main.c
```

### File: CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.16)
set(COMPONENTS main)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(keys_custom)
```

### File: sdkconfig.defaults

```ini
CONFIG_IDF_TARGET="esp32s3"
CONFIG_ESPTOOLPY_FLASHMODE_DIO=y
CONFIG_ESPTOOLPY_FLASHFREQ_80M=y
CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_PARTITION_TABLE_OFFSET=0x8000
CONFIG_ESP_CONSOLE_NONE=y
# CONFIG_SPIRAM is not set
# CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE is not set
# CONFIG_SECURE_BOOT is not set
# CONFIG_SECURE_FLASH_ENC_ENABLED is not set
```

These settings control the local build. They do not change the factory bootloader, flash encryption, eFuses or security state because **only the application BIN is uploaded**. The baseline does not provision encryption or modify device security. Its console is disabled because a physical UART header is not documented. No USB MIDI device is created by this scaffold.

### File: partitions.csv

Copy the complete [partition reference](partitions.csv) from this repository. It has the factory slot offsets, including App0 at `0x10000` and App1 at `0x210000`. This is an input to compilation and size checking, not an instruction to upload a new partition table.

### File: main/CMakeLists.txt

```cmake
idf_component_register(SRCS "main.c" INCLUDE_DIRS "."
    REQUIRES app_update esp_driver_gpio esp_psram)
```

### File: main/main.c

This original scaffold uses a **custom** recovery gesture: hold both encoder switches for at least one second while this application is running. It also works early in `app_main`. It is not the official Key0/five-click gesture and does not replace the newer bootloader gesture.

```c
#include <stdint.h>
#include "driver/gpio.h"
#include "esp_err.h"
#include "esp_ota_ops.h"
#include "esp_partition.h"
#include "esp_system.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

static esp_err_t enter_uploader(void)
{
    const esp_partition_t *target = esp_partition_find_first(
        ESP_PARTITION_TYPE_APP, ESP_PARTITION_SUBTYPE_APP_OTA_1, NULL);
    if (target == NULL || target->address != 0x210000 ||
        target->size != 0x100000) {
        return ESP_ERR_NOT_FOUND;
    }
    esp_err_t result = esp_ota_set_boot_partition(target);
    if (result == ESP_OK) {
        esp_restart();
    }
    return result;
}

void app_main(void)
{
    const gpio_config_t switches = {
        .pin_bit_mask = (UINT64_C(1) << 13) | (UINT64_C(1) << 38),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_ENABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_DISABLE,
    };
    if (gpio_config(&switches) != ESP_OK) {
        return;
    }
    TickType_t held_since = 0;
    int tracking_hold = 0;
    for (;;) {
        TickType_t now = xTaskGetTickCount();
        if (gpio_get_level(GPIO_NUM_13) == 0 &&
            gpio_get_level(GPIO_NUM_38) == 0) {
            if (!tracking_hold) {
                held_since = now;
                tracking_hold = 1;
            }
            if ((TickType_t)(now - held_since) >= pdMS_TO_TICKS(1000)) {
                (void)enter_uploader();
                tracking_hold = 0;
            }
        } else {
            tracking_hold = 0;
        }
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

The only persistent write this scaffold requests is the public OTA API's App1 boot selection, after a deliberate button hold. That selection persists in `otadata`; it is different from the newer bootloader's one-boot override. App1 manages the upload and subsequent return to App0. A missing or invalid App1 must produce a recoverable application error in a real project; add a visible status indication before distributing your firmware.

The scaffold does not initialize sound, LEDs, keyboard scanning, USB or BLE. Do not present it as finished musical firmware. Extend its loop or run a frequently scheduled recovery task before optional feature tasks, and handle errors explicitly. It cannot recover failures before `app_main` or a completely stalled CPU. Older devices retain that limitation.

## Build without touching a device

From `keys_custom` in the activated ESP-IDF terminal:

```text
idf.py set-target esp32s3
idf.py app
idf.py size
```

The file to upload is **`build/keys_custom.bin`**. The `app` target builds the application; a full `idf.py build` may additionally generate bootloader, partition-table and OTA-data images. Do not upload system images. Do not run `idf.py flash`, `app-flash`, `erase-flash`, `merge-bin` or production provisioning commands for this workflow. These instructions use the already installed App1 uploader.

PowerShell size check:

```powershell
$appImage = Get-Item -LiteralPath '.\build\keys_custom.bin'
if ($appImage.Length -gt 0x200000) { throw 'App0 exceeds its 2 MiB slot' }
Get-FileHash -LiteralPath $appImage.FullName -Algorithm SHA256
```

Cross-platform size and header check, using the activated environment's Python:

```text
python -c "from pathlib import Path; p=Path('build/keys_custom.bin'); b=p.read_bytes(); assert len(b)<=0x200000 and b[0]==0xe9; print(len(b), 'bytes')"
python -m esptool --chip esp32s3 image_info --version 2 build/keys_custom.bin
```

The magic byte and size checks catch common wrong-file mistakes; they do not prove image compatibility. Check the `image_info` target is ESP32-S3. Preserve the build's SDK version and file hash with every release.

## Enable external RAM after recovery is proven

In `idf.py menuconfig`, enable external SPI RAM support, select **Octal Mode**, auto-detect the device and initially use 40 MHz. Equivalent defaults for a fresh build are:

```ini
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MODE_OCT=y
CONFIG_SPIRAM_TYPE_AUTO=y
CONFIG_SPIRAM_SPEED_40M=y
CONFIG_SPIRAM_BOOT_INIT=y
CONFIG_SPIRAM_IGNORE_NOTFOUND=y
# CONFIG_SPIRAM_MEMTEST is not set
```

The ignore-not-found option lets the application reach its recovery handler if RAM is not detected. The application must check available PSRAM before allocating samples and fall back to a small internal-memory mode on failure. Do not place required startup data, task stacks or recovery code in PSRAM. The physical PSRAM capacity is not established by this guide; discover it at runtime instead of assuming 8 MB or 16 MB.

Changing `sdkconfig.defaults` does not overwrite an existing `sdkconfig`. Use `menuconfig` for the existing project or regenerate configuration in a separate clean build directory, retaining your previous working settings.

## Add features using public components

Use [the hardware contract](HARDWARE_CONTRACT.md) for electrical behavior and [the validation checklist](VALIDATION.md) before uploading. For I2S, use ESP-IDF 5.3.1's standard-mode driver. For USB MIDI, add an appropriate TinyUSB device component and MIDI descriptors; neither USB CDC nor USB Serial/JTAG is USB MIDI. For BLE MIDI, choose a public implementation of the BLE MIDI service and pin its dependency version. Those optional component integrations are not part of the no-dependency scaffold's build claim.

For an algorithmic synthesizer, implement your own oscillators and processing before the I2S output. For a sample-based instrument, add a compatible LittleFS component and SF2 playback engine: **SF2 is required for uploaded sample banks because App1 supports only executable BIN and SF2 files**. Do not design a direct WAV upload or arbitrary asset loader around an endpoint that does not support it. Convert source audio to SF2 on the host computer.

Do not mix Arduino 2.x legacy I2S calls with the ESP-IDF 5.x standard-mode driver in the same port. If you choose Arduino instead, the board package and every optional library require their own pinned versions and build validation.

## Validation status

See [VALIDATION.md](VALIDATION.md) for the exact compilation checks and the hardware work that remains. Successful compilation is not evidence of successful device boot or upload.

## Public SDK references

- [ESP-IDF v5.3.1 build system](https://docs.espressif.com/projects/esp-idf/en/v5.3.1/esp32s3/api-guides/build-system.html): project files, component dependencies and defaults.
- [ESP-IDF v5.3.1 OTA API](https://docs.espressif.com/projects/esp-idf/en/v5.3.1/esp32s3/api-reference/system/ota.html): partition selection and image validation.
- [ESP-IDF v5.3.1 standard I2S](https://docs.espressif.com/projects/esp-idf/en/v5.3.1/esp32s3/api-reference/peripherals/i2s.html): public audio driver interfaces.
- [ESP-IDF v5.3.1 external RAM](https://docs.espressif.com/projects/esp-idf/en/v5.3.1/esp32s3/api-guides/external-ram.html): allocation and startup considerations.
