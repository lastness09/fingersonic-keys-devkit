# Independent firmware validation and handoff

## What this documentation release establishes

The GPIO assignments, matrix index mapping, I2S signal configuration, partition layout and App1 entry policy were checked against project interface definitions and the manufacturer's instructions. The upload request shape was checked against the official Keys web updater. No proprietary product source or implementation file is included in the public kit.

The ESP-IDF build scaffold is an original example of public APIs. Its compilation results are recorded below. This audit did not connect to a physical instrument, install firmware, alter eFuses or erase device storage. Compilation and API checks do not establish hardware compatibility or prove that a particular App1 version accepts a custom image.

## Compilation checks

The documentation's fenced project files must build with ESP-IDF v5.3.1 and no private dependencies. Check the application fits 2 MiB, the image target is ESP32-S3, the partition table has exactly the published slot offsets and the optional OPI PSRAM configuration is recognized. Generated system images are build by-products and must not be distributed as installation inputs.

Audit performed on October 19, 2026:

| Check | Result |
| --- | --- |
| Documentation extraction | Project files were extracted directly from the fenced snippets in BUILD_FROM_SCRATCH.md; no product implementation files were copied |
| SDK and compiler | Installed ESP-IDF v5.3.1, Xtensa GCC 13.2.0, esptool 4.10.0 |
| Application build and size report | Passed for the internal-RAM recovery scaffold, including the documented component dependencies |
| App0 BIN size | 204,256 bytes; below the 2,097,152-byte slot limit |
| Image inspection | ESP32-S3 chip ID 9; 16 MB flash header; DIO, 80 MHz; checksum and image validation hash passed |
| Local validation BIN SHA-256 | `0323ea4e98c36ac8ab4ad44bdb93347dc35729f663c16ff187f38efe3d5001f9` |
| Optional PSRAM settings | Separate configure-only check passed: Octal Mode, 40 MHz, ignore-not-found enabled and boot memory test disabled; no physical RAM test was performed |
| Partition reference | Slot offsets, sizes, non-overlap and 16 MB boundary checked |
| Source boundaries | Build component paths contain the public SDK and the newly created scaffold, with no dependency on the private product project |
| Publication checks | English-only documents, local documentation links, explicit file allowlist and credential scan checked |

The installed SDK's embedded version string is `v5.3.1-dirty`; local differences include unused SDK example shell scripts and an unused NimBLE submodule, outside this scaffold's built components. This is a successful local build, not a claim that a pristine upstream checkout was independently rebuilt here. Install the pinned public release and retain your own build transcript. Binary hashes can differ with build metadata and environment. The hash above identifies the local audit artifact only; no firmware BIN is distributed by this documentation kit.

USB MIDI, BLE MIDI, LED output, key scanning, audio synthesis, SF2 playback, actual upload and recovery were not hardware-tested in this audit. Optional PSRAM configuration was checked without compiling or running a PSRAM-enabled application. These remain acceptance tasks for the developer and device owner.

## Gaps corrected by this audit

| Previous gap | Supplied information |
| --- | --- |
| A GPIO list without a complete build starting point | Pinned SDK, new project tree, every scaffold file, commands and artifact selection |
| An ambiguous legacy Arduino version note | ESP-IDF baseline independent of that note; optional libraries require their own version checks |
| Key-index swaps without a ready contact table | All four input pairs and six columns with swaps already applied |
| A scanner could accept signals from the wrong column | Drive/release ownership validation, isolated columns and a per-key contact event model |
| Audio format omitted important bus assumptions | Master TX, Philips timing, signed stereo PCM, pin polarity and unused MCLK/DIN |
| Generic OTA examples could destroy recovery | Dedicated App1 constraint and explicit prohibition on alternating-slot updates from App0 |
| A filesystem mount could silently destroy sounds | LittleFS label/format distinction, no automatic formatting, application-scoped persistent settings |
| Recovery instructions lacked an independent implementation starting point | Newly written public-API scaffold with a documented custom physical gesture |
| Compiling was easy to confuse with hardware verification | Separate compilation record, actual hardware checklist and honest untested-feature labels |

## Before the first upload

- Record the actual device's production date and which recovery method it supports. Do not infer this from purchase date.
- Obtain the official updater and understand how to restore official App0. Download or prepare any official recovery asset using the supported tool before leaving your normal network. Keep your previous working custom BIN and its hash.
- Check that the file is the application's own BIN, targets ESP32-S3 and fits `0x200000` bytes. A ZIP, ELF, merged BIN, filesystem image, bootloader or partition table is the wrong file.
- Review the firmware for writes to App1, partition tables, eFuses, whole-flash erase and automatic filesystem formatting. Do not use generic alternating-slot OTA from App0.
- Ensure a physical App1 gesture is serviced before optional initialization. For older units, compilation alone cannot guarantee recovery from a boot failure; accept that risk before experimenting.
- Prepare the upload on the host, then connect to the actual device hotspot using the official tool's connection instructions. Keep power stable and avoid repeated upload attempts while the first is processing.

## Hardware acceptance tests

| Area | Test | Pass condition |
| --- | --- | --- |
| Recovery | Invoke the custom physical gesture before enabling sound or storage, then again while the application is idle | App1 hotspot appears; a known-good App0 can be uploaded and started |
| New bootloader | On a supported production unit, hold Encoder2 through power-on | App1 is reachable independently of App0; do not intentionally brick an older unit to test this |
| Encoders | Rotate both knobs in both directions; press, hold and release each switch | Stable counts and direction, no rotation-induced button events, no phantom holds |
| Key mapping | Press each key individually from left to right and inspect both contact states | Indices 0-23 occur once each and the published mapping matches the physical keys |
| Velocity | Slow/fast presses, partial presses, held contacts and repeated notes | One Note On per press; monotonic velocity behavior; the chosen incomplete-contact policy is documented |
| Release | Release one contact and then both; repeat with several held keys | One Note Off when both contacts are released, no stuck notes or cross-triggering |
| LEDs | Walk one dim pixel from 0 through 23, then test red/green/blue | Key/pixel order and GRB interpretation are correct, no unintended full-brightness output |
| I2S | Begin with silence, then a quiet stereo test signal, then silence | Correct pitch, clean start/stop, no clipping, verified channel routing |
| USB MIDI | Connect to a host MIDI monitor; send notes and CCs, unplug and reconnect | Enumerates as MIDI and clears stuck notes; no repeated hotplug crash |
| BLE MIDI | Pair with a compatible host, play notes, disconnect and reconnect | Standard MIDI messages are received and active-note state is cleared after disconnect |
| Optional storage | Boot with missing or unreadable SF2 and with a failed filesystem mount | Application reaches its recovery handler and does not format the sound-bank partition |
| Sample-bank workflow | For a sample-based application, upload an SF2 prepared from host-side recordings and exercise its supported presets/zones | App0 reads SF2 correctly; unsupported banks fail gracefully; no direct WAV/MP3 or arbitrary asset upload is assumed |
| Optional PSRAM | Check detected capacity and simulate application allocation failure | Small internal-memory fallback and recovery continue to work |
| Combined load | Exercise polyphony, dense key input, LEDs, USB and BLE together for at least 10 minutes | No watchdog reset, repeated underruns, stuck notes or loss of recovery responsiveness |
| Restore | Return to official App0 through App1; restore sounds separately if changed | Official instrument behavior returns without changing system partitions |

Scope tests to enabled features. A MIDI-only controller does not need to implement a synthesizer or SF2 player. Power, charging and battery-level management need no application driver because the independent chip provides them.

## Required deliverables from an AI coding assistant

1. A complete independent project, with buildable source written for the user's requested behavior. It must not refer to unavailable Fingersonic headers, classes, private libraries or repositories.
2. Exact SDK and optional dependency versions, reproducible build commands, configuration files and a description of any assumptions.
3. The App0 BIN, byte size, SHA-256 hash and target chip information. No factory bootloader or system flashing package.
4. A control map, recovery gesture, upload instructions and a record of which tests above actually ran, on what hardware.
5. A clear distinction between compiler-verified, API-reviewed, host-simulated and hardware-tested behavior. Untested features remain labeled untested.

The owner chooses the license and distribution terms for their independent firmware. Publishing this hardware guide does not publish Fingersonic's official firmware or require access to its private source.
