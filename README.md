# ES3C35P

A hardware reference and working ESPHome configuration for the **LCDwiki
ES3C35P** — the 3.5" ESP32-S3 display board with the USB-C connector, sold as
`ES3C35P` (with speaker) or `ES3C35P-NS` (without).

The panel and the touchscreen both work under ESPHome with **no external
component and no C++**. Everything needed to drive them is in this repository.

## The board

| | |
|---|---|
| MCU | ESP32-S3, 8MB octal PSRAM, 16MB flash |
| Panel | ST77922, quad SPI, 320x480 native portrait, RGB565 |
| Touch | Sitronix controller integrated with the panel, I²C `0x55` |
| USB | Native USB-Serial/JTAG, USB-C |
| Also on board | ES8311 audio codec, SC8002B amplifier, microphone, microSD (4-bit SDIO), WS2812 RGB LED, lithium battery charging |

Vendor page: <https://www.lcdwiki.com/3.5inch_ESP32-S3_Display>

**If you bought this as something else, you are in the right place.** The same
board is sold rebadged with no mention of LCDwiki or ES3C35P — on Amazon as
*"Hosyond ESP32-S3 Touchscreen Module, 3.5" 240x320 IPS LCD EPS32 Display with
WiFi Bluetooth Capacitive Touch Screen for Arduino IoT Projects"*
([B0H28X8SQ4](https://www.amazon.com/dp/B0H28X8SQ4)). **The 240x320 in that
title is wrong** — the same listing's own specifications say 320x480 RGB565,
and that is what the panel does.

## Getting a picture on it

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
    dimensions: {width: 320, height: 480}
    color_order: RGB
    invert_colors: true
    draw_rounding: 4
    <<: !include st77922-init-sequence.yaml

touchscreen:
  - platform: st7123
    id: touch
    address: 0x55
    reset_pin: GPIO48
    display: lcd
```

The backlight is **GPIO41, active HIGH**, and `mipi_spi` has no backlight
support — drive it separately with a `ledc` output.

The panel is native portrait. For a 480x320 landscape UI use `rotation: 270`
**in the `lvgl:` block** — rotation on the display block is rejected outright
when LVGL is present, and 90 comes out mirrored because this controller
honours MADCTL MV but ignores MX.

**Then physically unplug the board.** The LCD reset line is tied to CHIP_PU, so
a reset over USB does not reset the panel. Coming off other firmware it will
stay black with a completely clean log until it gets one real power-on reset.
This is the single most likely reason a first flash appears to do nothing.

Requires **ESPHome 2026.8.2 or later** — the `st7123` touch platform landed
2026-07-02.

## What is here

| Path | |
|---|---|
| [`docs/HARDWARE.md`](docs/HARDWARE.md) | The reference: pinout, panel parameters, ESPHome contract, gotchas, prior art. Every claim graded by how it was established. |
| [`esphome/st77922-init-sequence.yaml`](esphome/st77922-init-sequence.yaml) | The panel's 56-entry initialisation sequence, with its provenance. |
| [`esphome/es3c35p-diag.yaml`](esphome/es3c35p-diag.yaml) | A layered bring-up diagnostic. Each stage disambiguates the next, so one boot tells you which half is broken. |
| [`esphome/es3c35p-lvgl.yaml`](esphome/es3c35p-lvgl.yaml) | A minimal LVGL configuration: landscape orientation, rotated touch, correct buffer sizing. |

## Why the grading

Vendor material for this board contains errors. The pin table prints UART0
backwards. The spec PDF gives the wrong touch I²C address. Two byte-different
copies of that PDF, both stamped with the same version and release date,
disagree with each other about I²S data direction.

So [`docs/HARDWARE.md`](docs/HARDWARE.md) records how each claim was
established — **probed** on a physical unit, decoded from the board's own
**firmware**, taken from **vendor** material, or **inferred** — next to the
claim itself. Where sources conflict, the document says which one the hardware
agreed with.

## Factory firmware

The 16MB factory image is **not** in this repository. It is LCDwiki's compiled
firmware and they grant no licence to redistribute it. Published instead: its
sha256, the byte offsets every `firmware`-graded claim was decoded at, and the
init table itself — which LCDwiki also publish openly in their Arduino
examples. That is enough to verify a dump of your own, and not enough to
reconstruct theirs.

**If you have one of these boards and have not flashed it yet, dump it now.**
See
[Factory firmware](docs/HARDWARE.md#factory-firmware). That window does not
reopen.

## Upstream

A `mipi_spi` model for this board is proposed at
[esphome/esphome#19011](https://github.com/esphome/esphome/pull/19011), with
its docs row at
[esphome/esphome.io#7341](https://github.com/esphome/esphome.io/pull/7341). The
10ms SLPOUT settle that this configuration works around is filed as
[esphome/esphome#19010](https://github.com/esphome/esphome/issues/19010).
