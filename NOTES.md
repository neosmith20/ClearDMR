# ClearDMR Session Notes

## Session 3 — 2026-05-01: Web CPS Read/Write Hardware Validation

### What We Accomplished

### 1. Web Serial radio read stabilized
- Normal radio operating mode using USB CDC / Web Serial
- VID/PID observed: `0x1FC9` / `0x0094`
- Port settings: `115200 8N1`
- Direct `R` requests are used
- `C 00` / `C FE` init commands are skipped before reads
- `R` responses are strict `R`-framed:
  `[0] = 0x52`
  `[1..2] = uint16 BE payload length`
  `[3..n] = payload`
- Reader uses `R` sync scanning because `0x52` often arrives as a separate 1-byte chunk

### 2. Radio info confirmed
- Radio info request:
  `52 09 00 00 00 00 00 00`
- Confirmed response is `R`-framed
- Confirmed `radioType 10` for DM-1701 path

### 3. Boot text write test hardware-proven
- Isolated page: `docs/cps/test.html`
- Boot text addresses:
  `0x7540 = line 1, 16 bytes`
  `0x7550 = line 2, 16 bytes`
- Boot text uses printable ASCII with `0xFF` padding
- Write requires:
  `fresh radio read`
  `saved backup`
  `isolated Write Boot Text Only button`
  `immediate readback verify`
- Hardware tests passed:
  `WRFS904 -> WRFS905`
  `WRFS905 -> WRFS904`
  verified after page refresh and radio power-cycle

### 4. STM32 write path confirmed
- Do not use `X 04` for STM32 boot text writes
- `X 04` can ACK but does not modify flash
- Proven working path is sector overlay:
  `X 01` prepare 4 KB sector
  `X 02` overlay changed bytes
  `X 03` erase/program sector
- This is used for boot text and boot mode test writes

### 5. Boot display mode confirmed
- Address: `0x7518`
- Size: `1 byte`
- `0x00 = Picture`
- `0x01 = Text`
- No checksum
- Confirmed using official CPS and ClearDMR test page:
  `Official CPS Picture -> ClearDMR reads Picture / 0x00`
  `Official CPS Text -> ClearDMR reads Text / 0x01`
- Test-page-only boot mode write succeeded

### 6. Main CPS write safety
- Main CPS full `Write to Radio` has been disabled/blocked for now
- Reason: full-codeplug write is not hardware-proven yet
- Isolated write paths remain only on `docs/cps/test.html`

### 7. Callsign write test hardware-proven
- Isolated page: `docs/cps/test.html`
- Callsign field:
  `offset 0x00E0`
  `length 8 bytes`
  `encoding printable ASCII`
  `padding 0xFF`
- Confirmed examples:
  `W0PWR   = 57 30 50 57 52 FF FF FF`
  `TEST123 = 54 45 53 54 31 32 33 FF`
- Proven STM32 write path:
  `X 01` prepare 4 KB sector
  `X 02` overlay bytes at `0x00E0`
  `X 03` erase/program sector
- Required guardrails:
  `fresh radio read`
  `saved backup`
  `ASCII-only validation`
  `max 8 characters`
  `pad unused bytes with 0xFF, never 0x00`
  `write only 0x00E0-0x00E7`
  `immediate readback verify`
  `mark session stale after success and require another fresh read before any further write`

### 8. DMR ID write hardware-proven in `docs/cps/test.html`
- Offset:
  `0x00E8-0x00EB`
- Length:
  `4 bytes`
- Encoding:
  `packed decimal / BCD-like byte order`
- Valid range:
  `1 to 16776415`
- Confirmed examples:
  `3214246 -> 03214246 -> 03 21 42 46`
  `3214247 -> 03214247 -> 03 21 42 47`
  `3214288 -> 03214288 -> 03 21 42 88`
- Encoding rule:
  convert the DMR ID to 8 decimal digits
  left-pad with `0`
  store two digits per byte in forward byte order
- Hardware-proven isolated DMR ID write path:
  `3214246 -> 3214247 -> 3214246`
  `03 21 42 46 -> 03 21 42 47 -> 03 21 42 46`
  ClearDMR write succeeded using STM32 sector overlay:
  `X 01 prepare sector`
  `X 02 overlay 4 bytes at 0x00E8`
  `X 03 erase/program sector`
  immediate byte-for-byte readback verify passed
  OpenGD77 CPS also confirmed the DMR ID changed correctly
  DMR ID was reverted successfully after test
  fresh read / power-cycle persistence confirmed if tested
- Isolated DMR ID write support exists only on `docs/cps/test.html`
- Do not enable full-codeplug write
- Do not integrate DMR ID write into the main CPS yet

### Current Safe Status

- `docs/cps/test.html`:
  hardware-proven boot text write
  hardware-proven boot mode write
  hardware-proven callsign write
  hardware-proven DMR ID write
  DMR ID inspector remains visible
  isolated DMR ID write support only
  read-only custom-data inspectors for boot image and satellite TLE blocks
  read-only voice prompt flash header / TOC inspection and backup tooling
  test-page-only candidate MCU ROM spot-dump tooling
- `docs/cps/index.html`:
  read/open/save/edit only
  full radio write disabled
- Main CPS integration should wait until isolated write paths are boringly stable

### Read-Only Mapping Status

- Boot image area:
  confirmed as OpenGD77 custom-data block type `0x01`
  lives inside the codeplug custom-data area at file `0x1EE60-0x1FFFF`
  observed block header at `0x1FAE4`
  observed payload start at `0x1FAEC`
  boot image payload size is `1024` bytes (`128x64 / 8`)
  test page now exposes read-only preview and raw export
- Satellite data area:
  confirmed as OpenGD77 custom-data block type `0x03`
  lives inside the same codeplug custom-data area
  observed block header at `0x1F074`
  observed payload start at `0x1F07C`
  firmware satellite custom block size is `2520` bytes
  test page now exposes read-only preview and raw export
- OpenGD77 custom-data block layout inside the 128 KB codeplug:
  start `0x1EE60`
  length `4512` bytes
  header starts with ASCII `OpenGD77`
  block header format appears to be:
  `byte 0 type`
  `bytes 1-3 reserved / zero`
  `bytes 4-7 uint32 LE length`
  `bytes 8.. payload`
  observed block sequence in multiple codeplugs:
  `0x1EE6C type 2 melody len 512`
  `0x1F074 type 3 satellite len 2520`
  `0x1FA54 type 4 day theme len 64`
  `0x1FA9C type 5 night theme len 64`
  `0x1FAE4 type 1 boot image len 1024`
  `0x1FEEC uninitialised 0xFF area`
- Voice prompts:
  firmware constants on STM32 place the current header at raw SPI flash `0xAF400`
  legacy header candidate remains `0x100000`
  header is `8` bytes and the colour-radio TOC is `368 * 4 = 1472` bytes
  test page now exposes read-only header / TOC inspection and active-block backup
- Secure registers:
  firmware USB handler exposes flash security registers at CPS access area `0x0A`
  current test-page-only backup target is start `0x00000000`, length `768` bytes
  this is now the smallest extra read-only protocol backup path in the test lab
- MCU ROM:
  firmware USB handler exposes CPS access area `5` for MCU ROM reads
  main proven JS helper remains unchanged
  source start address is `0x00000000`
  STM32 full length target is `1048576` bytes
  test page currently exposes only small read-only candidate dumps for vector table and app header
- Backup SRAM / RTC backup registers:
  still pending
  current CPS read handler does not expose BKPSRAM or RTC backup registers directly

### Next Steps

- Add a verbose/debug logging toggle to reduce normal console spam
- Continue testing boot text, boot mode, callsign, and DMR ID isolated writes
- Run callsign edge cases:
  `A -> 41 FF FF FF FF FF FF FF`
  `full 8-character callsign -> no padding`
  `revert to original callsign and verify`
- Run DMR ID edge cases:
  `3214246 -> 03 21 42 46`
  `3214247 -> 03 21 42 47`
  `3214288 -> 03 21 42 88`
  `lowest valid 1 -> 00 00 00 01`
  `highest valid 16776415 -> 16 77 64 15`
- Compare picture/text backups to document changed bytes
- Validate secure-register read-only backup on hardware using area `0x0A`
- Validate voice prompt flash header / TOC reads on hardware and confirm whether the active header is current or legacy on each target radio
- Validate area `5` MCU ROM reads on hardware before attempting any larger ROM backup flow
- Determine whether backup SRAM / RTC backup registers need firmware-side read exposure before they can be inspected safely
- Later integrate boot text, boot mode, and callsign controls into main CPS with the same guardrails
- Do not enable full-codeplug write until separately validated
- Do not integrate DMR ID write into the main CPS until the isolated test path is boringly stable

## Session 1 — 2026-04-24: CMake Migration & CI

### What We Accomplished

### 1. CMake Build System (`firmware/CMakeLists.txt`)
Replaced the Eclipse CDT managed build (`.cproject`) with a clean CMake setup that:
- Targets STM32F405VGTx (Cortex-M4F, `-mfpu=fpv4-sp-d16 -mfloat-abi=hard`)
- Preserves the bootloader flash offset — application starts at `0x800C000` via `STM32F405VGTX_FLASH.ld`
- Handles all 8 build variants via `-DPLATFORM=` and boolean options (see below)
- Applies per-file `-O0` override on `codec_interface.c` (all other sources use `-Os`)
- Embeds the git short hash at configure time for version strings in `hotspot.c`, `usb_com.c`, `menuFirmwareInfoScreen.c`
- Automates codec blob generation so a fresh clone builds without manual steps

### 2. Multi-Variant Presets (`firmware/CMakePresets.json`)
Eight named presets covering all shipped configurations:

| Preset | Platform | 10W | Japanese |
|---|---|---|---|
| `mduv380` | MDUV380 | — | — |
| `mduv380-10w` | MDUV380 | yes | — |
| `mduv380-ja` | MDUV380 | — | yes |
| `mduv380-10w-ja` | MDUV380 | yes | yes |
| `dm1701` | DM1701 | — | — |
| `dm1701-ja` | DM1701 | — | yes |
| `rt84` | RT84 | — | — |
| `rt84-ja` | RT84 | — | yes |

Build a variant locally:
```sh
cmake --preset mduv380          # configure
cmake --build --preset mduv380  # build
```

### 3. GitHub Actions CI (`.github/workflows/build.yml`)
- All 8 presets run in parallel (`fail-fast: false`)
- Installs `gcc-arm-none-eabi`, `libnewlib-arm-none-eabi`, `cmake` via apt
- Uploads `WaveForge_*.bin` + `WaveForge.hex` per variant as 30-day artifacts
- Triggers on push/PR to `main` when `firmware/**` or the workflow itself changes

### 4. Artifact Naming
All output binaries use the `WaveForge_` prefix (e.g. `WaveForge_MDUV380.bin`).
The raw `WaveForge.bin`/`.hex` are also uploaded for debug/flashing convenience.

---

## Session 2 — 2026-04-24: Codec Donor Workflow, Flashing, and WaveForge Branding

### 1. Codec Donor Workflow

The real AMBE codec blob cannot be committed to the repo, but can be extracted from
a donor device (radio running official or OpenGD77 firmware) using `codec_cleaner`:

```sh
# Extract from a donor binary (e.g. an official factory .bin you legally own)
firmware/tools/codec_cleaner.Linux -e <donor_firmware.bin> \
    firmware/application/source/linkerdata/codec_bin_section_1.bin
```

Once extracted, place the file at:
`firmware/application/source/linkerdata/codec_bin_section_1.bin`

Then re-run CMake configure — the generated `codec_bin_generated.S` will pick up the
real blob automatically (it uses an absolute path injected at configure time).

**Verification:** After flashing, key up on a DMR channel. You should hear encoded
audio. The zero-filled placeholder firmware links and boots but produces no audio.

### 2. Flashing via usbipd / WSL2

Because the ST-Link interface is a USB device, it must be forwarded into WSL2 before
`openocd` or `st-flash` can see it. One-time setup (Windows PowerShell, elevated):

```powershell
# Install usbipd-win if not already present (winget or GitHub releases)
winget install usbipd

# List attached USB devices to find the ST-Link bus ID (e.g. 2-3)
usbipd list

# Bind once (persists across reboots)
usbipd bind --busid 2-3

# Attach to WSL each session
usbipd attach --wsl --busid 2-3
```

Inside WSL2 (after attach):

```sh
# Confirm the device appears
lsusb | grep -i stlink

# Flash using st-flash (install: sudo apt install stlink-tools)
st-flash write firmware/build/mduv380/WaveForge_MDUV380.bin 0x800C000

# Or via OpenOCD (config file exists at firmware/MDUV380_firmware.cfg)
openocd -f firmware/MDUV380_firmware.cfg \
        -c "program firmware/build/mduv380/WaveForge_MDUV380.bin verify reset exit"
```

**Note:** The flash address `0x800C000` is the application start — do not flash to
`0x8000000` or you will overwrite the bootloader.

### 3. WaveForge Branding Changes (committed as `825e2fa`)

Three source changes in `firmware/application/source/user_interface/`:

| File | Change |
|---|---|
| `uiSplashScreen.c` | Boot splash `"OpenGD77"` → `"WaveForge"` |
| `menuFirmwareInfoScreen.c` | Firmware info radio model label → `"WaveForge"` |
| `menuFirmwareInfoScreen.c` | Credits roll: added `"-- WaveForge --"` and `"Alex W0PWR"` |

### 4. `version.h` — Semantic Version Replaces Build Timestamp (committed as `d4d41ac`)

Added `firmware/application/include/version.h` defining a single source of truth for
the firmware version:

```c
#define WAVEFORGE_VERSION_MAJOR 1
#define WAVEFORGE_VERSION_MINOR 0
#define WAVEFORGE_VERSION_PATCH 0
#define WAVEFORGE_VERSION_STRING "1.0.0"
```

`menuFirmwareInfoScreen.c` was updated to display `WAVEFORGE_VERSION_STRING` on the
firmware info screen instead of the old `__DATE__`/`__TIME__` build timestamp. This
gives reproducible, human-readable version strings that survive repeated builds.

**To bump the version:** edit the three `_MAJOR`/`_MINOR`/`_PATCH` defines in
`version.h` and the `_STRING` macro. Increment patch for bug fixes, minor for new
features, major for breaking changes.

### 5. GUI Flasher — Idea Noted

Discussed the idea of a small GUI flasher tool (Windows/macOS/Linux) that wraps the
`st-flash` / OpenOCD command line so end-users can flash ClearDMR without a terminal.
Key UX goals: auto-detect connected ST-Link, present a file picker for the `.bin`,
enforce the correct `0x800C000` flash address, and show progress. Captured in the
roadmap below.

---

## The Codec Binary Situation

This is the trickiest part of the build. Read carefully.

### What it is
`codec_bin_section_1.bin` is a **proprietary DMR AMBE codec binary blob** that gets
placed at a fixed absolute flash address `0x807537C`. The linker script creates a
custom section `.codec_bin_section_1` mapped to that exact address.

### The problem with the original source
The original `firmware/application/source/dmr_codec/codec_bin.S` uses a path
**relative to the Eclipse build output directory**:
```asm
.incbin "../application/source/linkerdata/codec_bin_section_1.bin"
```
This breaks for any non-Eclipse build location.

### How we solved it
CMake generates a replacement `.S` file at configure time with an **absolute path**
(`firmware/build/<preset>/codec_bin_generated.S`), and runs `codec_cleaner -C` to
produce a zero-filled placeholder blob if the real blob is absent.

### Where to find the real blob
- **Placeholder (what CI uses):** auto-generated by `firmware/tools/codec_cleaner.Linux -C`
  → produces an all-zeros `codec_bin_section_1.bin` (audio will not work, but firmware links)
- **Real blob location in source tree:** `firmware/application/source/linkerdata/codec_bin_section_1.bin`
  (this directory has a `readme.txt` explaining it; the file itself is **not committed** — it's gitignored or simply absent)
- **How to get the real blob:** Run `codec_cleaner -C` in the linkerdata directory, OR
  extract it from an official OpenGD77 / factory firmware binary using `codec_cleaner`
  in extract mode. The `codec_cleaner` tools for all platforms live at:
  - Linux:   `firmware/tools/codec_cleaner.Linux`
  - Windows: `firmware/tools/codec_cleaner.exe`
  - macOS:   `firmware/tools/codec_cleaner` (no extension)

### Key constraints — do not break these
- Fixed flash address `0x807537C` — hardcoded in the linker script. Never move it.
- `codec_interface.c` must be compiled with `-O0`. It contains timing-sensitive
  codec interface code that breaks under optimization.
- The generated `.S` file (`codec_bin_generated.S`) lives in the **build directory**,
  not the source tree. The original `codec_bin.S` in the source tree is **not used**
  by the CMake build — it remains there only as reference.

---

## Roadmap

### UX / tooling
- [ ] **GUI flasher tool** — cross-platform desktop app (Tk, Electron, or similar) that
  wraps `st-flash`/OpenOCD: auto-detects ST-Link, file picker for `.bin`, enforces
  `0x800C000` flash address, shows progress. Removes the usbipd/terminal barrier for
  end users who just want to flash and go.

### Near-term (build system / infrastructure)
- [ ] **Verify CI passes end-to-end** — confirm all 8 matrix jobs go green
  (toolchain install + configure + build + artifact upload).
- [ ] **Real codec blob in CI** — currently CI uses the zero-filled placeholder.
  Supply the real blob via a GitHub Actions secret + a pre-configure script to
  produce flashable artifacts from CI.
- [ ] **Remaining 10 Eclipse build variants** — the 18 Eclipse configs include
  hardware sub-variants (V1/V2/V4/V5) and debug builds not yet covered by the 8
  CMake presets. Each needs additional `PLATFORM_VARIANT_*` defines wired through
  CMake options.
- [ ] **Flash/debug CMake targets** — OpenOCD config (`MDUV380_firmware.cfg`) exists
  but isn't wired into CMake. Add `flash` and `debug` targets using the usbipd/WSL
  workflow documented above.
- [ ] **Windows/macOS local build docs** — the toolchain file assumes the
  cross-compiler is on `PATH`. Document or auto-detect common install paths.
- [ ] **Update README** — still references the original OpenGD77 project.

### Software / firmware
- [ ] **CPS (Code Plug Software)** — update or rebuild the Code Plug Software for
  ClearDMR. The existing CPS references OpenGD77 branding and may have protocol
  assumptions tied to upstream. Goal: a ClearDMR-branded CPS that works out of the
  box with ClearDMR firmware.
- [ ] **FreeRTOS update** — bring FreeRTOS to the current upstream release. Audit
  the BSP tick config, heap scheme, and any OpenGD77-specific patches before merging.
- [ ] **STM32 HAL driver updates** — update STM32 HAL/LL drivers to a recent STM32CubeF4
  release. Watch for conflicts in the clock config and USB driver.
- [ ] **Python tools — Python 3 audit** — inventory all Python scripts under
  `firmware/tools/` and the CPS. Port anything still on Python 2 and pin minimum
  version to 3.10+.

### Community
- [ ] **Community outreach** — post to relevant DMR/ham radio forums and subreddits
  to attract contributors. Write a CONTRIBUTING.md, set up GitHub Discussions, and
  document the codec donor workflow so newcomers can build a working image.

---

## Key File Locations

| What | Where |
|---|---|
| CMake entry point | `firmware/CMakeLists.txt` |
| Build presets | `firmware/CMakePresets.json` |
| Toolchain file | `firmware/cmake/arm-none-eabi.cmake` |
| Linker script (flash) | `firmware/STM32F405VGTX_FLASH.ld` |
| CI workflow | `.github/workflows/build.yml` |
| Codec blob assembly | `firmware/application/source/dmr_codec/codec_bin.S` (original, unused by CMake) |
| Codec cleaner tools | `firmware/tools/codec_cleaner{,.Linux,.exe}` |
| Codec blob placeholder dir | `firmware/application/source/linkerdata/` |
| Generated codec `.S` (build-time) | `firmware/build/<preset>/codec_bin_generated.S` |
