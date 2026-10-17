# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Open Source Cartridge Reader (OSCR): Arduino Mega 2560 firmware that dumps cartridge ROMs and saves straight to an SD card, with no PC needed. The README notes the project has moved to https://github.com/oscartreader. Layout:

- `Cart_Reader/`: the firmware sketch (GPL v3). This is where nearly all code work happens.
- `sd/`: database files copied to the SD card root (name/CRC32/size/save type per game). `.gitattributes` makes them **CRLF**, and the firmware depends on that, so don't normalize their line endings. `sd/README.md` documents the per-file formats.
- `hardware/`: KiCad PCBs for the main board and the adapters (CC BY 4.0).
- `tools/oscr_tool/`: a Windows PowerShell helper for managing the portable release.
- `pinout.ods`: the pin mapping each core uses.

## Build

There are no tests. "Testing" means compiling. CI (`.github/workflows/ci.yml`) does it like this:

```sh
arduino-cli core install arduino:avr
arduino-cli lib install "SdFat" "Adafruit SSD1306" "Adafruit GFX Library" "Adafruit BusIO" \
  "U8g2" "Adafruit NeoPixel" "RotaryEncoder" "Etherkit Si5351" "RTClib" "FreqCount"
cd Cart_Reader
arduino-cli compile --fqbn arduino:avr:mega --warnings all --build-property compiler.cpp.extra_flags="-DGITHUB_CI"
```

`Config.h` as committed does **not** compile, because no hardware version is selected. To build, you uncomment one of `HW1`–`HW5`/`SERIAL_MONITOR` and choose cores. CI edits `Config.h` with `sed` for each matrix entry (HW3 and HW5):
- **ALL_DEFAULTS**: the default cores together (GBX, N64, NES, SMS, MD, SNES). It runs when `Config.h`, `Cart_Reader.ino`, `OSCR.*` or `ClockedSerial.*` change.
- **Single core**: the default cores are commented out and only the changed module's `ENABLE_*` is enabled. CI finds a file's core by grepping the **first 10 lines** of each `.ino` for its `ENABLE_*` name, so keep the `#ifdef ENABLE_XXX` guard near the top of a module.

To check one core locally, do what CI does: set a HW version, disable the defaults, enable the core, compile. Then revert `Config.h` before committing. A committed `ALLOW_UNSAFE_CONFIG` is a hard error under `-DGITHUB_CI`.

Flash/SRAM is tight. Enabling too many cores builds a firmware that shows a blank screen or "Press Button" message. Releases build with `-mcall-prologues` and a newer avr-gcc (see `bi-monthly_release.yml`).

## Architecture

The Arduino build concatenates every `.ino` into one translation unit, with `Cart_Reader.ino` first. Functions and globals are therefore shared across modules without headers.

- **`Config.h`**: the user-facing feature toggles (`HWx`, `ENABLE_<CORE>`, hardware modules like `ENABLE_VSELECT`/`ENABLE_CLOCKGEN`/`ENABLE_RTC`, firmware options). Its tail derives display/input defines from the HW version (`ENABLE_LCD`/`ENABLE_OLED`/`ENABLE_SERIAL`, `ENABLE_ROTARY`, etc.).
- **`OSCR.h` / `OSCR.cpp`**: sanity-check `#error`s for incompatible configs, shared PROGMEM strings (`FSTRING_*`, which include `FSTRING_VERSION`, bumped on release), and the two central enums, both built conditionally on the `ENABLE_*` flags:
  - `enum CORES`: the current operating mode (`mode` global).
  - `enum SYSTEM_MENU`: main menu entries. `SYSTEM_MENU_TOTAL` must stay last.
- **`Cart_Reader.ino`**: the shared runtime. It contains `setup()`, the main menu (`mainMenu()` with the `modeOptions[]` PROGMEM array), output helpers (`print_Msg`/`println_Msg`/`display_Update`, which hide whether the output is an OLED, LCD or serial port), input/menu helpers (`question_box`, `pageMenu`, `wait`), SD/file helpers (`createFolderAndOpenFile`, etc.), CRC/database lookup, and `loop()`. `loop()` dispatches on `mode` to each core's `xxxMenu()`.
- **One `.ino` per system** (`NES.ino`, `SNES.ino`, `GB.ino`, …), each wrapped in `#ifdef ENABLE_XXX`. Each defines `setup_XXX()`, which configures the pins, and `xxxMenu()`, plus its read/write routines. Some depend on another core: SFM/SV/ST/GPC need `ENABLE_SNES`, CPS3 needs FLASH8 and FLASH16, and the SNES "Flash repro" menu needs both `ENABLE_FLASH` and `ENABLE_FLASH8` (otherwise it shows "please enable module in Config.h").
- **`FLASH.ino`**: the flashrom/repro writers. `writeByte_Flash`/`readByte_Flash` switch on the global `mapping` (0 = flash adapter, 1 = SNES LoROM, 2 = HiROM, 3 = PLCC32, 122/222/124/224/142/242 = multi-chip SNES repros). For mapping 1 and 2, the CFI write and `verifyFlash` use the faster `addressPorts_Flash`/`writePorts_Flash`/`readData_Flash` helpers. 32-bit shifts like `addr >> 15` compile into slow loops on AVR, so hot loops avoid them.

**Adding a core** touches several files: a new `XXX.ino` guarded by its flag, an `ENABLE_XXX` block in `Config.h`, entries in `CORES` and `SYSTEM_MENU` in `OSCR.h`, a menu string plus `modeOptions[]` entry and a `mainMenu()` case in `Cart_Reader.ino`, a `loop()` dispatch case, and the `ALL_CORES` list in `ci.yml`.

## Conventions (from `Cart_Reader/README.md`)

- Access pins directly through the AVR registers (`DDRx`, `PORTx`, `PINx`), not `digitalWrite`/`pinMode`. `pinout.ods` has the mapping.
- Put every string in flash: use `F("...")` for literals, and declare menu items as `const char X[] PROGMEM` with a `const char* const menuOptionsX[] PROGMEM` array. Reuse the `FSTRING_*` constants (e.g. `FSTRING_RESET` as the last menu item).
- Never call the display library directly. Go through the `print_Msg`/`println_Msg`/`display_*` helpers so that every output backend (OLED, LCD, serial) keeps working.
- The sources (`Config.h`, `*.ino`) use CRLF line endings. Keep them CRLF when editing.
- Commit messages typically start with a system tag, e.g. `[NES] ...`, `[VSMILE] ...`.
