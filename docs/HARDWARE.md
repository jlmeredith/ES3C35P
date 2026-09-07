# LCDwiki ES3C35P — hardware reference

3.5" ESP32-S3 display board, sold as **ES3C35P** (with speaker) or **ES3C35P-NS**
(without). Vendor page: <https://www.lcdwiki.com/3.5inch_ESP32-S3_Display>.

**It is also sold rebadged.** On Amazon it appears as *"Hosyond ESP32-S3
Touchscreen Module, 3.5" 240x320 IPS LCD EPS32 Display with WiFi Bluetooth
Capacitive Touch Screen for Arduino IoT Projects"*
([B0H28X8SQ4](https://www.amazon.com/dp/B0H28X8SQ4)), which names neither
LCDwiki nor ES3C35P anywhere. The 3.5" option ships with a speaker, so it is
the `ES3C35P` rather than the `ES3C35P-NS`. **Ignore the 240x320 in that
title** — the same listing's own specifications say 320x480 RGB565, which is
what the hardware does.

Every claim below carries a grade for how it was established. Vendor material
for this board contains errors — a swapped UART row and a wrong touch I²C
address — so knowing where a number came from matters as much as the number.

| Grade | Meaning |
|---|---|
| **probed** | Measured over USB, or exercised under ESPHome with the expected physical result. **On one unit.** Nothing here is a sample of boards, so a claim that depends on panel lot or assembly variant may not generalise. |
| **firmware** | Decoded from the board's own factory image (sha256 `cb90e3d2…9971cd`, 16MB). Byte offsets are given so a dump of your own can be checked against them. Shows what the shipped code intends, not what the copper does. |
| **datasheet** | From the ST77922 controller datasheet (Sitronix Preliminary v0.1, [published by Espressif](https://dl.espressif.com/AE/esp-iot-solution/ST77922_SPEC_V0.1.pdf)). Page numbers are given. |
| **vendor** | From the LCDwiki wiki, spec PDF, schematic, or LCDwiki's published Arduino examples. |
| **inferred** | Deduced from other facts. Treat as a hypothesis. |

The factory image itself is not distributed here — see
[Factory firmware](#factory-firmware).

---

## Specifications

| Property | Value | Grade |
|---|---|---|
| MCU | ESP32-S3 (QFN56) rev v0.2, Xtensa LX7 dual-core @ 240MHz | **probed** |
| PSRAM | 8MB embedded **octal** (OPI), 80MHz | **probed** — boots with `mode: octal, speed: 80MHz, ignore_not_found: false` |
| Flash | 16MB, manufacturer `0x5e`, device `0x4018` | **probed** |
| Crystal | 40MHz | **probed** |
| USB | Native USB-Serial/JTAG, VID `0x303A` PID `0x1001`, USB-C | **probed** — no CH340, no external bridge |
| Serial port | Enumerates as a CDC device (`/dev/cu.usbmodem*` on macOS) | **probed** |
| WiFi MAC OUI | `7c:e8:b1` (Espressif) | **probed** |
| Panel | ST77922, quad SPI, **320x480 native portrait**, RGB565 | **probed** |
| Touch | Sitronix controller integrated with the panel, I²C `0x55` | **probed** |
| Audio codec | ES8311, I²C `0x18` | **probed** — answers the bus scan |
| Amplifier | SC8002B | vendor |
| Microphone | On board, routed through the ES8311 | vendor — untested |
| Battery | External lithium connection with onboard charge management; sense on GPIO8 (ADC1_CH7) | vendor — untested |
| Partition table | nvs, otadata, app0/app1 3MB each, ffat 9.875MB, coredump 64K | **firmware** — parsed at flash `0x8000` |

ESPHome validates either PSRAM mode without complaint, so a config that
compiles proves nothing. `ignore_not_found: false` plus a successful boot is
what establishes octal mode.

The factory build settings are recorded verbatim in the image's own FQBN string
(`app0.bin 0x001ff0`), which is the authoritative source for the PSRAM question:

```
FlashSize=16M  PSRAM=opi  PartitionScheme=app3M_fat9M_16MB
FlashMode=qio120  USBMode=hwcdc  CPUFreq=240
```

---

## Pinout

No line here has been metered. Every line graded **probed** was driven under
ESPHome and produced the expected physical effect.

| Function | GPIO | Grade |
|---|---|---|
| LCD QSPI CS | 10 | **probed** |
| LCD QSPI CLK | 12 | **probed** |
| LCD QSPI D0 / D1 / D2 / D3 | 11 / 13 / 14 / 9 | **probed** |
| LCD reset | **tied to `EN` / CHIP_PU** — no independent reset line | **probed** — see [Reset](#the-panel-needs-a-real-power-on-reset) |
| LCD backlight | 41, **active HIGH** | **probed** — driven to both rails; HIGH lights the panel |
| LCD tearing effect (TE) | 42 | **firmware** — the TE ISR registers GPIO42 in the factory image; absent from every vendor pin table. Not exercised here, since ESPHome has no TE support |
| Touch I²C SDA / SCL | 38 / 39 | **probed** |
| Touch reset | 48 | **probed** |
| Touch interrupt | 47 | vendor — untried |
| microSD (4-bit SDIO) CLK / CMD | 5 / 4 | vendor |
| microSD D0 / D1 / D2 / D3 | 6 / 7 / 2 / 3 | vendor |
| Audio I²S MCLK / BCLK / LRCLK | 17 / 18 / 21 | vendor |
| Audio I²S DOUT / DIN | 15 / 16 | vendor — untested, see below |
| Amplifier shutdown (SC8002B) | 1 | vendor — **active LOW**, see below |
| RGB LED, single-wire addressable | 40 | **probed** — WS2812, `channel_colors: GRB` |
| Battery sense (ADC1_CH7) | 8 | vendor |
| BOOT button | 0 | **probed** |
| UART0 TX / RX | **43 / 44** | corrected, see below |
| Free expansion | 45 / 46 | vendor |

**Touch, the ES8311 codec and the 4-pin expansion header share one physical I²C
bus** (**probed** — one scan on GPIO38/39 answers for both `0x18` and `0x55`;
the expansion header is **vendor**). GPIO38 and GPIO39 are only available for
other use if both touch and audio are unused.

### Corrections to vendor material

- **The retail listing's resolution is wrong in its own title.** The Amazon
  title says 240x320; the specification bullets on the same page say 320x480
  RGB565. The panel is 320x480 — CASET and RASET in the board's own firmware
  say so, and it draws correctly at that size.
- **UART0 is printed backwards.** The vendor pin table gives `RXD0(IO43)` /
  `TXD0(IO44)`. In ESP32-S3 silicon U0TXD is GPIO43 and U0RXD is GPIO44. Use
  the silicon assignment.
- **The touch address is `0x55`, not `0x38`.** The spec PDF's `0x38` is wrong —
  nothing responds there. A bus scan finds `0x18`, `0x28` and `0x55`, and
  `0x55` answers every read.
- **GPIO15 is I²S output and GPIO16 is input**, on every source that can be
  checked: the wiki pin table, the spec PDF the vendor currently publishes
  (sha256 `67c343bf…eed2e3`, stamped `V1.0 / First Release / 2025-06-14`), and
  the merged xiaozhi port (`AUDIO_I2S_GPIO_DOUT GPIO_NUM_15`,
  `AUDIO_I2S_GPIO_DIN GPIO_NUM_16`). A second copy of that PDF, stamped
  identically, reverses them. Neither direction has been exercised here, so
  take 15/16 as out/in and confirm before relying on it.
- **Amplifier shutdown is active LOW.** The pin is named SHUTDOWN, the vendor
  table says "low level enable", and the xiaozhi ESP-IDF port for this board
  passes `pa_inverted = true`. Do not drive it high to enable.

---

## Panel: ST77922

320x480 native portrait, quad SPI, RGB565.

### Initialisation sequence

The full sequence is in
[`esphome/st77922-init-sequence.yaml`](../esphome/st77922-init-sequence.yaml).
Under `init_sequence:` that file holds 59 items: a leading `delay 120ms`, the
56 vendor commands (`0xF1` through `RASET`), then `11h SLPOUT` and a
`delay 200ms`.

It is a 63-entry `lcd_init_cmd_t` array in the factory image at `app0.bin
0x0b3ec4`, reached through an `st77922_vendor_config_t` at `0x0b3e98` =
`{init_cmds = 0x3c1a3ec4, init_cmds_size = 63, use_qspi = 1}`. A second copy
sits at `app0.bin 0x0eb06c` in a different compilation unit. The two decode to
identical `{cmd, data, delay_ms}` tuples but are **not** byte-identical: each
entry's `data` pointer differs, because each copy addresses its own parameter
blobs (`app0.bin 0x100670–0x1007bb` and `0x100b2c–0x100c77`, no overlap). 126
of the 1008 bytes differ, all of them pointer bytes.

Each 16-byte record is `{cmd, data*, data_bytes, delay_ms}`, so **the dump
carries the per-entry delays too**: entry 58 is `11h` with `delay_ms` 120 and
entry 63 is `35h` with `delay_ms` 20, and every other entry is 0.

The same table is published openly in LCDwiki's own Arduino examples
([ydedox/st77922](https://github.com/ydedox/st77922),
`Example_01_Simple_test`). What that source uniquely provides is the **second
variant**: the file carries two complete 63-entry tables, an active one and a
commented-out one directly above it, for the two panel variants of this
reference design. This board runs the **commented-out** variant (54 of 56
entries match) with two parameters taken from the active one — `0x71` and
`0xBE`, the 19th and 21st entries counting the array from one. Having both
tables is what makes that decomposition provable.

**A similar table at `app0.bin 0x0b43d0` is not this panel's.** It is the
upstream `esp_lcd_st77922` driver's built-in default for a 532x300 panel —
CASET/RASET `00 00 02 13` / `00 00 01 2b` — loaded only in the branch taken
when `init_cmds` is NULL, which on this board it never is. Its parameter blobs
live in the same `.data` segment as this panel's but occupy a separate range;
resolving every `data` pointer in all three arrays shows no blob is shared
between any two of them.

Six of the last seven vendor entries — `21h INVON`, `29h DISPON`, `2Ch RAMWR`,
`3Ah COLMOD`, `36h MADCTL`, `35h TEON` — are absent from the published YAML.
`11h SLPOUT` is kept, moved to the end; see
[the appended tail](#the-appended-slpout-tail-and-what-it-guarantees).

ESPHome does not simply append an equivalent for all six. It appends `3Ah`,
`21h`/`20h` and `29h`; it writes `36h MADCTL` at runtime from
`reset_params_()` rather than inside the sequence; it issues `2Ch` on every
write; and it has **no `35h TEON` equivalent at all**. It also rejects a `3Ah`
inside a user sequence — unless that sequence contains a page-select (`0xFE`
or `0xFF`), which switches the check off. Express the rest through
`invert_colors:`, `color_order:` and the LVGL rotation.

### Panel parameters

| Parameter | Value | Grade |
|---|---|---|
| Geometry | 320 x 480, no offset | **probed** — CASET `00 00 01 3F`, RASET `00 00 01 DF` |
| Colour order | **RGB** (MADCTL `0x00`) | **probed** — corroborated by the vendor driver's `{0x36, {0x00}, 1, 0}` |
| Inversion | **`invert_colors: true` is required** | **probed** |
| Pixel format | RGB565. The vendor writes COLMOD `0x01`; the panel also accepts ESPHome's `0x55` | **probed** |
| Draw alignment | **4 pixels** | **datasheet** — §2.1 lists "GIP + Dual-Gate driving". Confirmed harmless on hardware, but not confirmed *necessary*: the test card repaints the whole frame, so `draw_rounding` never changes the window written (see below) |
| Clock | **40MHz probed working.** The controller's write-clock ceiling is 62.5MHz; the factory firmware runs 80MHz, 28% over it | **datasheet** for the ceiling, **probed** at 40MHz |
| Landscape | **`rotation: 270`** in the `lvgl:` block — MADCTL `0xA0` | **probed** — see [MADCTL and rotation](#madctl-and-rotation) |

**`draw_rounding` is not established by the test card.** ESPHome's `fill()`
marks the entire band dirty, and 320 and 480 are both already multiples of 4,
so the window written is byte-identical at `draw_rounding` 1, 2 or 4 and no
partial redraw ever occurs. The value comes from the dual-gate architecture and
matches the Freenove PR; the photograph only shows that 4 does no harm.

**40MHz is the fastest rate ESPHome can reach at or below 62.5MHz, not the only
one.** ESP32 SPI rates are an 80MHz/N ladder with a 5% tolerance, so 26.67, 20,
16MHz and downward are all selectable — there is simply nothing between 40 and
80MHz. The SPI write is not the bottleneck anyway; see
[Performance](#performance).

### MADCTL and rotation

**The panel honours MADCTL MV (axis swap) and ignores MX (mirror X).** Setting
`rotation: 90`, which makes ESPHome write MADCTL `0x60` (MV|MX), produces a
clean and correctly proportioned landscape image that is **mirrored** — a
transpose without the accompanying flip, which is exactly the result of MV
landing and MX being dropped.

That contradicts the xiaozhi port, which asserts the swap is impossible —
`static_assert(!DISPLAY_SWAP_XY, "ST77922 does not support swapping the X and Y
axes")` in `main/boards/lcdwiki-es3c35p/lcdwiki-es3c35p.cc`. It can. (The
Freenove PR makes no such statement; it simply declares the mirrors only.)

**`rotation: 270` is the correct landscape value**, and it is what this
configuration uses. It writes MADCTL `0xA0` (MV|MY) — MV transposes the axes
and MY supplies the flip that MX could not — and renders a correctly oriented,
unmirrored 480x320 landscape image. `0xA0` is also the value the vendor's own
Arduino driver computes for landscape (`LCD_Set_Rotation` case 3).

The same driver's case 1 computes `0x44` with no MV bit while still swapping
width and height. That is not a contradiction once MX is known to be
non-functional — it is what a driver looks like when it was written against a
part that ignores the bit.

**Hardware rotation is all-or-nothing, which is why a shipped model should
still choose software.** ESPHome's `DriverChip.has_hardware_transform` is an
*equality* test against the full `{mirror_x, mirror_y, swap_xy}` set, not a
subset test. A model cannot therefore offer the working 270 without also
offering 90, and 90 on this silicon is the mirrored screen above — with nothing
in the log to explain it, because QSPI writes are write-only and a controller
ignoring a MADCTL bit is indistinguishable from one honouring it. Declaring
only the mirrors costs a rotate buffer and some CPU per flush and is always
correct, which is what the upstream model does.

`model: CUSTOM` carries the full set, so a `CUSTOM` display with
`lvgl: rotation: 270` does take the hardware path — that is what this
repository's configuration uses. Adding `transform: disabled` to the display
block forces software rotation instead.

---

## Touch: ESPHome's `st7123` platform

The integrated controller is a Sitronix part whose register map matches
ESPHome's in-tree `st7123` touchscreen platform exactly: 16-bit big-endian
addresses `0x0001` / `0x0005` / `0x0009` / `0x0010` / `0x0014`, 7-byte stride,
`0x80` valid bit, `0x3F` coordinate mask. **No external component and no C++
are needed.**

```yaml
touchscreen:
  - platform: st7123
    address: 0x55
    reset_pin: GPIO48
    display: lcd          # no interrupt_pin — keeps the 50ms poller alive
```

Read off the controller itself at setup:

| Property | Value | Note |
|---|---|---|
| Max touches | **5** | Read from `ST7123_REG_MAX_TOUCHES`. The driver's own default is 10, so this is the chip's answer, not a default. |
| Raw X / Y range | **320 / 480** | Matches panel native. No scaling needed. |
| Status register | `0` on first read | ESPHome's 5ms reset pulse plus 30ms settle is sufficient on this board — the driver never reports `Failed to read status register`. No ST7123 datasheet is cited here, so treat any specific reset-timing minimum as unestablished. |

Live touches map straight through with no calibration; `x_raw` runs one count
above `x`, which is the driver's own coordinate mapping rather than an offset
to correct.

### A cold-boot I²C scan will not find `0x55`

This is not a fault. The `i2c` component scans during its own setup, which runs
before the touchscreen's `setup()` releases `reset_pin` GPIO48 — so on a cold
boot the touch controller is still held in reset. After a soft reboot GPIO48 is
still high from the previous run and the same scan finds everything.

| Boot type | Scan finds |
|---|---|
| Cold power-on | `0x18` only |
| Soft reboot / OTA | `0x18`, `0x28`, `0x55` |

Do not read a cold-boot scan as evidence about the touch address. The bus also
needs recovery on every boot (`Performing bus recovery` → `Recovery: bus
successfully recovered`), consistent with the un-reset touch controller holding
SDA low.

**`0x28` is undocumented.** It appears and disappears exactly with `0x55`, so
it belongs to the touch controller (**probed**). What it is for is **not
established** — no datasheet consulted here names it. It is not needed for
operation.

---

## ESPHome

Verified against **ESPHome 2026.8.2**; every file and line reference below is
to that version. The `st7123` touch platform first shipped in **2026.7.0**, so
that is the floor for the touch half.

There is no `ST77922` display model and no `st77922` touchscreen platform in
ESPHome. Neither is needed:

- **Display** — `mipi_spi` accepts `model: CUSTOM` with `bus_mode: quad` and a
  user `init_sequence`. Data only, no C++.
- **Touch** — the in-tree `st7123` platform drives the controller as shipped.

A `mipi_spi` model for this board is proposed upstream as
[esphome/esphome#19011](https://github.com/esphome/esphome/pull/19011).

### Display block

```yaml
spi:
  - id: lcd_bus
    type: quad
    clk_pin: GPIO12
    data_pins: [GPIO11, GPIO13, GPIO14, GPIO9]

display:
  - platform: mipi_spi
    id: lcd
    spi_id: lcd_bus
    model: CUSTOM
    bus_mode: quad
    cs_pin: GPIO10
    data_rate: 40MHz
    dimensions:
      width: 320
      height: 480
    color_order: RGB
    invert_colors: true
    draw_rounding: 4
    <<: !include st77922-init-sequence.yaml
```

`mipi_spi` has **no backlight dimming or runtime control**. Its `enable_pin:`
can hold GPIO41 on, but anything more — brightness, turning the panel off —
needs a separate `ledc` output plus a `monochromatic` light.

`mipi_spi` has **no TE support**. The factory firmware builds a whole GPIO42
tearing-effect subsystem with an ISR and two semaphores; ESPHome has no
equivalent, so expect tearing on full-frame redraws.

### LVGL

**Rotation goes in the `lvgl:` block, never the display block.** With LVGL
present, ESPHome rejects any **non-zero** `rotation:` on a display (`rotation: 0`
validates and is simply redundant), and rejects `lambda:`, `pages:`,
`auto_clear_enabled: true` and `show_test_card: true` outright
(`components/lvgl/__init__.py`, `final_validation`).

```yaml
lvgl:
  displays: [lcd]
  touchscreens: [touch]
  rotation: 270       # 480x320 landscape from a 320x480 panel
  buffer_size: 25%
```

Three things follow from that, and each of them is a trap if you assume
otherwise:

1. **LVGL takes the hardware rotation path.** `model: CUSTOM` is a `DriverChip`
   with no declared defaults, so its transform set is the full
   `{mirror_x, mirror_y, swap_xy}` and `has_hardware_transform` is true. LVGL
   therefore selects `ROTATION_HARDWARE` and calls `display->set_rotation()`
   itself, which lands in `mipi_spi`'s `reset_params_()` and rewrites MADCTL at
   runtime. To force software rotation instead, add `transform: disabled` to
   the display block — the compile log then reads `INFO LVGL will use software
   rotation`.

2. **The touchscreen needs no `transform:` and no `calibration:`.** The
   touchscreen component caches the display's dimensions in its own
   `call_setup()` at priority `DATA` (600), while `LvglComponent` runs at
   `PROCESSOR` (400) — so it caches the **un-rotated** 320x480 and reports points in the
   panel's native frame. LVGL's input-device callback then applies
   `rotate_coordinates()`. **Adding `swap_xy` here rotates twice.** If touches
   come out transposed, the rotation direction is wrong, not the touch config.

3. **`draw_rounding` propagates automatically.** LVGL takes the `max()` of the
   display's value and its own, so `draw_rounding: 4` does not need repeating
   in the `lvgl:` block.

**Prefer a small `buffer_size` over a large one.** 8MB of PSRAM makes a full
307,200-byte framebuffer trivially affordable, but `LvglComponent::setup()`
only attempts internal RAM when the requested fraction is small — so asking for
100% guarantees the draw buffer lands in PSRAM, where every CPU blend is
slower. Drawing is the bottleneck on this board, not the bus, so the buffer
wants to be fast to draw into rather than large. 25% is 76,800 bytes, fits
internal SRAM, and LVGL only ever redraws dirty rectangles anyway. It falls
back to PSRAM on its own if internal allocation fails.

### The appended SLPOUT tail, and what it guarantees

`mipi_spi` appends `delay(0)`, `11h SLPOUT`, `delay(10)`, `29h DISPON` to every
sequence. **The `delay(0)` is not a no-op.** `mipi_spi` records `millis() + 120`
before running the sequence, and a zero delay sleeps until that instant — so
the appended SLPOUT is already guaranteed to land at least 120ms after the
reset.

That is the requirement the datasheet actually states. Sleep Out cannot be sent
for 120ms after reset (p.60 note 7, and SWRESET p.181), but only **5ms** is
required between SLPOUT and the next command (p.190). ESPHome allows 10ms
there, which is twice the requirement.

So the framework's timing is datasheet-correct, and a 10ms post-SLPOUT wait is
**not** a cause of a black panel. This configuration still issues `11h` at the
end of its own `init_sequence` followed by `delay 200ms`, matching the margin
in the vendor's own table (`{0x11, ..., 0, 120}`) and making ESPHome's later
SLPOUT a no-op on an already-woken panel:

```yaml
  - [0x11]
  - delay 200ms
```

That is belt and braces, not a fix for a framework defect. **A black panel with
a clean log on this board is far more likely to be
[the missing power-on reset](#the-panel-needs-a-real-power-on-reset)**, which
produces an identical symptom and is not a timing problem at all.

The syntax needs a unit — `delay 200ms`, not `delay 200` — and the value is
capped at 255ms.

### COLMOD cannot be overridden, and does not need to be

With `bus_mode: quad` the schema forces `pixel_mode` to `16bit`, and the
framework appends `3Ah = 0x55` unconditionally. The in-source comment claiming
it appends only "if not already in the custom sequence" is wrong — it appends
either way.

This is not a problem. The panel renders correct RGB565 from `0x55`; the
vendor's `0x01` is one valid encoding rather than the only one. The in-tree
`ESP-VOCAT` model, in `models/st77916.py`, is a QSPI Sitronix panel driven
through the identical path.

---

## Gotchas

### The panel needs a real power-on reset

**After flashing a board that was running other firmware, physically unplug it.
A black screen with a clean log is this, not your init sequence.**

The LCD reset line is tied to `EN` / CHIP_PU. A reset over native
USB-Serial/JTAG — `esptool --after hard-reset`, an OTA, `esp_restart` — resets
the CPU **without toggling CHIP_PU**, so the panel is never hardware-reset. It
stays in whatever state the previous firmware left it. `SWRESET` does not
recover it.

Symptom: the init sequence completes with no error, `Setup display took 151ms`,
full frames clock out in 28ms, the backlight is confirmed lit — and the screen
is black. Nothing in the log is wrong, because SPI writes are write-only and
always "succeed".

**There is no log-side tell, and INVON is not one.** A panel that stays black
under `invert_colors: true` has *not* thereby proved that commands are failing
to arrive. Per the datasheet, sleep-in stops the DC/DC converter, the internal
oscillator and panel scanning (p.189), and `28h DISPOFF` blanks the output
independently — so a panel that received every command and is simply not
scanning renders black exactly like one that received none. Inversion only
changes what is scanned out, so it says nothing when nothing is being scanned.

Treat the black screen as ambiguous and settle it the cheap way: **power-cycle
first**, before touching the init sequence. Fix: unplug the USB-C cable, wait
~10s, plug it back in.

**This is a bring-up rule, not an operating rule.** Once the panel has had one
power-on init, soft reboots are fine — an OTA re-initialises it cleanly
(`rst:0xc RTC_SW_CPU_RST`) with the image still on screen afterwards.

### OTA plus a reset inside ~60s silently rolls back

Bootloader rollback is enabled. A freshly uploaded image is provisional until
the log reads `[I][safe_mode]: Boot seems successful`. Reset before that point
and the bootloader reverts to the previous image while `INFO OTA successful`
scrolls past — the upload genuinely succeeded, and the running firmware is
still the old one.

Diagnose by comparing the boot banner's `compiled on` timestamp against the
build's `build_time_str`. Wait for the safe-mode line before resetting.

### `esphome run` can upload a stale binary

`<name>.bin` and `firmware.ota.bin` in the same build directory are not always
the same build — a 38-minute gap between them has been observed. When a change
refuses to appear on the device, upload the OTA image explicitly:

```bash
esphome upload es3c35p-diag.yaml --device <ip> --file .esphome/build/<name>/build/firmware.ota.bin
```

Both configs here use the native ESP-IDF toolchain, which is the 2026.8.2
default for `esp32`. Its outputs are `.esphome/build/<name>/build/<name>.bin`
and `.../build/firmware.ota.bin`. `.pioenvs/` is the PlatformIO layout and does
not exist under this toolchain.

### Backlight polarity cannot be settled by eye

With the panel dark, a **lit** backlight still looks black. Settle polarity by
toggling GPIO41 between rails and watching for a change, not by asking whether
it glows.

### Long flash reads drop over native USB

**Read the 16MB image in 256KB chunks.** A single whole-image read drops
partway. Chunking is what completed here; a full-image read did not.

Do not read anything into the `--baud` value. This board's port is the
ESP32-S3's own USB-Serial/JTAG peripheral, i.e. a CDC virtual port whose line
rate the host records and the link never uses. esptool applies `--baud`
unconditionally, so the number can be set and appears to matter, but it does
not gate throughput — a 949KB application image writing and verifying in 6.1s
is ~155kB/s, roughly seven times what 230400 baud could carry. If long reads
drop for you, chunk them; changing the baud rate is not the lever.

### Do not run the logger at VERBOSE

`VERBOSE` prints the WiFi PSK in clear text and buries the display command dump
under per-byte I²C traffic. `DEBUG` keeps everything useful.

---

## Performance

At 40MHz with a full 320x480 framebuffer in PSRAM (307,200 bytes):

| Stage | Time |
|---|---|
| Lambda drawing into the buffer | ~146ms |
| SPI write of the full frame | ~28ms |
| **Total update** | **~177ms** |

ESPHome logs `display took a long time for an operation (178 ms), max is 50 ms`.

**The bottleneck is drawing, not the bus** — five times the SPI cost. Raising
the clock to 80MHz would buy ~14ms of a 177ms frame. The lever that matters is
drawing less per frame: partial updates, fewer `filled_rectangle` calls, or
LVGL, which only ever flushes dirty rectangles.

---

## Factory firmware

**The factory image is not in this repository and never has been.** It is
LCDwiki's compiled firmware and they grant no licence to redistribute it. What
is published instead is its identity and everything decoded from it:

- **sha256** `cb90e3d245a402ec9ea79cc464dfa15b4f31a349f39fda4e548a15117b9971cd`,
  16MB, Arduino-ESP32 core 3.2.0 on ESP-IDF v5.4.1, sketch compiled 2025-07-01.
  (The `Mar 28 2025` date in the app descriptor belongs to the prebuilt
  `arduino-lib-builder` library, not the sketch.)
- Every byte offset each **firmware**-graded claim was read at.
- The decoded init sequence, which is also published independently in LCDwiki's
  own Arduino examples.

Compare that hash against a dump of your own to know you have the same image
this document was written from.

### Dump yours before flashing anything

There is exactly one chance, and it does not reopen. Read it in 256KB chunks —
a single whole-image read drops partway:

```bash
for i in $(seq 0 63); do
  off=$(printf '0x%x' $((i * 0x40000)))
  esptool --port /dev/cu.usbmodem* read-flash $off 0x40000 "chunk-$(printf '%02d' $i).bin"
done
cat chunk-*.bin > stock-firmware-ES3C35P.bin
```

Then check the size is exactly 16777216 bytes and take its sha256.

### Writing it back

BOOT is GPIO0 and RESET acts on CHIP_PU — hold BOOT while tapping RESET to
force download mode.

```bash
esptool --port /dev/cu.usbmodem* write-flash \
  --flash-size keep --flash-mode keep --flash-freq keep \
  0x0 stock-firmware-ES3C35P.bin
```

`keep` preserves whatever the dump's own bootloader header already encodes. On
this image those bytes are `e9 04 02 4f`: **DIO**, 16MB, 80MHz — *not* the
`qio120` in the sketch's FQBN. The two are not in conflict; the ROM bootloader
reads the header in DIO and the application switches the bus afterwards.
Passing `keep` is what avoids having to care.

If your system `esptool` is broken, ESPHome bundles a working one — invoke it
through ESPHome's own interpreter as `python -m esptool`.

---

## Prior art

- **[ydedox/st77922](https://github.com/ydedox/st77922)** — a mirror of
  LCDwiki's own Arduino example set for this board (29 examples). The most
  useful single source. `Example_01_Simple_test` carries the complete driver in
  two files: `spi_dev.h` confirms the pinout independently (CS 10, BL 41, SCLK
  12, D0–D3 = 11/13/14/9, 80MHz, `SPI_MODE0`, 320x480), and `Simple_test.ino`
  carries **two** complete 63-entry init tables — an active one and a
  commented-out one — which is what makes this board's variant identifiable.
  (The per-entry delays are in the flash dump too; the Arduino source is not
  needed for those.)
- **[esphome/esphome#18411](https://github.com/esphome/esphome/pull/18411)** —
  Freenove FNK0104N support: a full ST77922 touch component plus a `mipi_spi`
  model at 320x480, `draw_rounding: 4`, 80MHz, and a 200ms settle delay. Closed
  unmerged on process grounds — `boards.py` is generated and must not be
  hand-edited, and a PR should touch one component — with no reviewer disputing
  the logic. Diffed in full rather than sampled: of the 54 command entries the
  two tables share, 45 are identical and **9 differ** (`0x70`, `0x90`, `0x91`,
  `0x92`, `0x93`, `0x96`, `0x97`, `0xBA`, `0x86`), and the FNK0104N table
  carries no `2Ah`/`2Bh` at all. It is a template rather than a drop-in.
- **[78/xiaozhi-esp32#2112](https://github.com/78/xiaozhi-esp32/pull/2112)** —
  a merged ESP-IDF port of this exact board at `main/boards/lcdwiki-es3c35p/`.
  Independent corroboration of the pinout and the `0x55` register map. It does
  **not** invert colours, which is wrong for this panel.
- **[espressif/arduino-esp32#12694](https://github.com/espressif/arduino-esp32/issues/12694)**
  — someone on the same N16R8 ST77922 board who erased the factory firmware and
  wanted working example code. **This is where the vendor Arduino examples
  surfaced**: ydedox posted <https://github.com/ydedox/st77922> there on
  2026-06-23 and the issue was closed the same day. Worth reading in full — it
  also points at Espressif's `ESP32_Display_Panel` and the `esp_lcd_st77922`
  IDF component.
- `components/mipi_spi/models/st77916.py` in the ESPHome tree — the closest
  in-tree precedent, and evidence that a QSPI Sitronix panel works through this
  exact framework path.

**Weight these correctly.** The vendor wiki, spec PDF and schematic are **one**
source with one author. The xiaozhi and Freenove ports are **one** lineage —
they share 46 init commands. The vendor Arduino examples are a separate leg:
same author as the wiki, but working code with real timings rather than prose.
The board's own factory binary is the only fully independent source.

---

## Upstream

- **[esphome/esphome#19011](https://github.com/esphome/esphome/pull/19011)** —
  a `mipi_spi` model for this board. Data only, no C++, one file: 320x480, RGB
  order, `invert_colors`, `draw_rounding: 4`, 40MHz, CS on GPIO10, and the init
  table. It cites LCDwiki's published Arduino source rather than the firmware
  dump. Docs row:
  [esphome/esphome.io#7341](https://github.com/esphome/esphome.io/pull/7341).

  The model declares `transforms={mirror_x, mirror_y}` only, so rotation is
  done in software. That is deliberate: software rotation is correct whether or
  not a given unit's controller performs the hardware swap, while declaring a
  transform the chip lacks breaks every user of the model.

- **[esphome/esphome#19010](https://github.com/esphome/esphome/issues/19010)** —
  the hard-coded 10ms after `mipi_spi`'s appended `SLPOUT`, filed with the
  vendor's own `{0x11, …, 120}` as evidence, plus the incorrect comment above
  the `PIXFMT` append in the same function.
