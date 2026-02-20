# VGA 20×15 Text Mode (Lucid V2 HDL)

This project implements a simple **VGA text-mode renderer** for **640×480 @ 60 Hz** using **Lucid HDL**. The visible region is divided into **20 columns × 15 rows** of character cells. Each cell displays an **8×8 glyph** (scaled up to 32×32 pixels) with independently selectable **foreground** and **background** colors.

This project is written in [Lucid V2 HDL](https://alchitry.com/tutorials/lucid-reference/) and is meant to run on [Alchitry Au](https://alchitry.com/boards/au/) FPGA.

## Credits and Attribution

This design is based on the original 2020 Lucid HDL work by **Ragul Balaji** (SUTD 50.002 Computation Structures).

- **Original author:** Ragul Balaji (2020)
- **Module concept:** FPGA VGA 20×15 Text Mode
- **Timing reference:** http://tinyvga.com/vga-timing/640x480@60Hz

If you plan to redistribute or reuse substantial parts of Ragul Balaji’s original implementation, follow any licensing or permission requirements included with the source you received.

## What This Module Does

The renderer generates:

- **HSYNC / VSYNC** signals for 640×480 @ 60 Hz timing
- **RGB output (3-bit total)**: 1 bit each for R, G, and B (8 colors total)
- A tile-based text output where each tile is a character cell

### Display Layout

- Visible resolution: **640×480**
- Cell size: **32×32 pixels**
- Grid: **20×15 cells** (300 total)
- Glyph size: **8×8 pixels**, scaled by 4× to fill each 32×32 cell

## VGA Timing Overview

The design counts through the full timing envelope, not just visible pixels.

### Horizontal (per scanline)

- Visible: 640
- Front porch: 16
- HSYNC pulse: 96
- Back porch: 48  
  Total: **800 pixel clocks per line**

### Vertical (per frame)

- Visible: 480
- Front porch: 10
- VSYNC pulse: 2
- Back porch: 33  
  Total: **525 lines per frame**

A ~25 MHz pixel tick is derived from the 100 MHz FPGA clock using a divider and edge detector.

## Key Signals

### `vga_mem_address[9]` (output)

Selects the current **text cell** being drawn.

The address is computed from the current pixel counters:

- `tile_x = hzcnt >> 5` (divide by 32)
- `tile_y = vtcnt >> 5` (divide by 32)
- `vga_mem_address = tile_x + (tile_y * 20)`

### Address range

- Visible cells: **0..299**
- 9 bits are used because 300 cells exceed 8-bit capacity (256)
- 9 bits support **0..511**

### Linear layout

- `0` = row 0 col 0 (top-left)
- `1` = row 0 col 1
- ...
- `19` = row 0 col 19
- `20` = row 1 col 0
- ...
- `299` = row 14 col 19 (bottom-right)

> During blanking (porches and sync), counters still run. If your memory only defines 0..299, return a default or blank value for out-of-range addresses.

### `vga_mem_data[16]` (input)

Provides **character + colors** for the selected cell.

### Bit layout

- `[7:0]` ASCII character code (fed into `vga_font_rom.asciicode`)
- `[10:8]` Background color (3-bit RGB, 1 bit per channel)
- `[13:11]` Foreground color (3-bit RGB, 1 bit per channel)
- `[15:14]` Unused / reserved (currently ignored by the renderer)

### Color encoding (3-bit RGB)

Each color field is `{R,G,B}` with 1 bit each:

| RGB | Color   |
| --- | ------- |
| 000 | Black   |
| 100 | Red     |
| 010 | Green   |
| 001 | Blue    |
| 110 | Yellow  |
| 101 | Magenta |
| 011 | Cyan    |
| 111 | White   |

### Per-pixel selection

The font ROM outputs a 1-bit `color` indicating whether the glyph pixel is on:

- If `font_rom.color == 1`, output **foreground** (`[13:11]`)
- Else, output **background** (`[10:8]`)

## Font ROM (`vga_font_rom`)

`vga_font_rom` is a combinational glyph ROM:

- Input: `asciicode` (8-bit), `row` (0..7), `col` (0..7)
- Output: `color` (1-bit pixel on/off)

Glyphs are stored as 64-bit bitmaps representing 8×8 characters.

Row and column are derived from pixel position:

- `row = (vtcnt >> 2) & 7`
- `col = (hzcnt >> 2) & 7`

This scales each glyph pixel to 4×4 screen pixels.

## Pin Mapping (Chosen Pins)

The original implementation notes the following chosen output pins:

- `led[0]` → `D49` (RED)
- `led[1]` → `D48` (GREEN)
- `led[2]` → `D2` (BLUE)
- `led[3]` → `D3` (HSYNC)
- `led[4]` → `D46` (VSYNC)

If your board routes these pins to a VGA or VGA-compatible resistor network, this mapping drives the display directly. If you are using a different board or connector, update the top-level pin mapping and constraints accordingly.

## RGB Bit Ordering Note

Some boards or adapter wiring route the color lines in BGR order instead of RGB. The module includes a bit swap:

- `vga_rgb[0] = sig_RGB[2]`
- `vga_rgb[1] = sig_RGB[1]`
- `vga_rgb[2] = sig_RGB[0]`

Adjust this mapping if your board’s wiring differs.

## 3-bit RGB (Not 3 Bits Per Channel)

This design outputs **3 bits total**, one bit each for R, G, and B:

- Total colors: **8**

VGA itself is analog. If you want more color depth (for example 3 bits per channel), you need:

- Wider color buses (for example 9 bits total), and
- A resistor ladder or DAC that converts those bits into analog voltage levels.

## Minimal Hardcoded Screen Test

To display a constant character and color everywhere, feed `vga_mem_data` from a combinational constant VRAM:

- ASCII `'A'` = `0x41`
- FG = white = `111`
- BG = black = `000`
- `[15:14] = 00`

Packed value:

- `vga_mem_data = 16b00_111_000_0100_0001`

## Files

- `vga_text_mode.luc` # VGA 20×15 Text Mode (Lucid HDL)

This project implements a simple **VGA text-mode renderer** for **640×480 @ 60 Hz** using **Lucid HDL**. The visible region is divided into **20 columns × 15 rows** of character cells. Each cell displays an **8×8 glyph** (scaled up to 32×32 pixels) with independently selectable **foreground** and **background** colors.

## Credits and Attribution

This design is based on the original 2020 Lucid HDL work by **Ragul Balaji** (SUTD 50.002 Computation Structures).

- **Original author:** Ragul Balaji (2020)
- **Module concept:** FPGA VGA 20×15 Text Mode
- **Timing reference:** http://tinyvga.com/vga-timing/640x480@60Hz

If you plan to redistribute or reuse substantial parts of Ragul Balaji’s original implementation, follow any licensing or permission requirements included with the source you received.

## What This Module Does

The renderer generates:

- **HSYNC / VSYNC** signals for 640×480 @ 60 Hz timing
- **RGB output (3-bit total)**: 1 bit each for R, G, and B (8 colors total)
- A tile-based text output where each tile is a character cell

### Display Layout

- Visible resolution: **640×480**
- Cell size: **32×32 pixels**
- Grid: **20×15 cells** (300 total)
- Glyph size: **8×8 pixels**, scaled by 4× to fill each 32×32 cell

## VGA Timing Overview

The design counts through the full timing envelope, not just visible pixels.

### Horizontal (per scanline)

- Visible: 640
- Front porch: 16
- HSYNC pulse: 96
- Back porch: 48  
  Total: **800 pixel clocks per line**

### Vertical (per frame)

- Visible: 480
- Front porch: 10
- VSYNC pulse: 2
- Back porch: 33  
  Total: **525 lines per frame**

A ~25 MHz pixel tick is derived from the 100 MHz FPGA clock using a divider and edge detector.

## Key Signals

### `vga_mem_address[9]` (output)

Selects the current **text cell** being drawn.

The address is computed from the current pixel counters:

- `tile_x = hzcnt >> 5` (divide by 32)
- `tile_y = vtcnt >> 5` (divide by 32)
- `vga_mem_address = tile_x + (tile_y * 20)`

### Address range

- Visible cells: **0..299**
- 9 bits are used because 300 cells exceed 8-bit capacity (256)
- 9 bits support **0..511**

### Linear layout

- `0` = row 0 col 0 (top-left)
- `1` = row 0 col 1
- ...
- `19` = row 0 col 19
- `20` = row 1 col 0
- ...
- `299` = row 14 col 19 (bottom-right)

> During blanking (porches and sync), counters still run. If your memory only defines 0..299, return a default or blank value for out-of-range addresses.

### `vga_mem_data[16]` (input)

Provides **character + colors** for the selected cell.

### Bit layout

- `[7:0]` ASCII character code (fed into `vga_font_rom.asciicode`)
- `[10:8]` Background color (3-bit RGB, 1 bit per channel)
- `[13:11]` Foreground color (3-bit RGB, 1 bit per channel)
- `[15:14]` Unused / reserved (currently ignored by the renderer)

### Color encoding (3-bit RGB)

Each color field is `{R,G,B}` with 1 bit each:

| RGB | Color   |
| --- | ------- |
| 000 | Black   |
| 100 | Red     |
| 010 | Green   |
| 001 | Blue    |
| 110 | Yellow  |
| 101 | Magenta |
| 011 | Cyan    |
| 111 | White   |

### Per-pixel selection

The font ROM outputs a 1-bit `color` indicating whether the glyph pixel is on:

- If `font_rom.color == 1`, output **foreground** (`[13:11]`)
- Else, output **background** (`[10:8]`)

## Font ROM (`vga_font_rom`)

`vga_font_rom` is a combinational glyph ROM:

- Input: `asciicode` (8-bit), `row` (0..7), `col` (0..7)
- Output: `color` (1-bit pixel on/off)

Glyphs are stored as 64-bit bitmaps representing 8×8 characters.

Row and column are derived from pixel position:

- `row = (vtcnt >> 2) & 7`
- `col = (hzcnt >> 2) & 7`

This scales each glyph pixel to 4×4 screen pixels.

---

## Pin Mapping (Chosen Pins)

The original implementation notes the following chosen output pins:

- `led[0]` → `D49` (RED)
- `led[1]` → `D48` (GREEN)
- `led[2]` → `D2` (BLUE)
- `led[3]` → `D3` (HSYNC)
- `led[4]` → `D46` (VSYNC)

If your board routes these pins to a VGA or VGA-compatible resistor network, this mapping drives the display directly. If you are using a different board or connector, update the top-level pin mapping and constraints accordingly.

## RGB Bit Ordering Note

Some boards or adapter wiring route the color lines in BGR order instead of RGB. The module includes a bit swap:

- `vga_rgb[0] = sig_RGB[2]`
- `vga_rgb[1] = sig_RGB[1]`
- `vga_rgb[2] = sig_RGB[0]`

Adjust this mapping if your board’s wiring differs.

## 3-bit RGB (Not 3 Bits Per Channel)

This design outputs **3 bits total**, one bit each for R, G, and B:

- Total colors: **8**

VGA itself is analog. If you want more color depth (for example 3 bits per channel), you need:

- Wider color buses (for example 9 bits total), and
- A resistor ladder or DAC that converts those bits into analog voltage levels.

## Minimal Hardcoded Screen Test

To display a constant character and color everywhere, feed `vga_mem_data` from a combinational constant VRAM:

- ASCII `'A'` = `0x41`
- FG = white = `111`
- BG = black = `000`
- `[15:14] = 00`

Packed value:

- `vga_mem_data = 16b00_111_000_0100_0001`

## Files

- `vga_text_mode.luc` : VGA timing, text-cell addressing, and color selection
- `vga_font_rom.luc` : ASCII glyph ROM (8×8 bitmap font)

## Notes

- The visible-area bounds have been fixed in your version, so the renderer now uses the full **640×480** visible region.
- This module expects `vga_mem_data` to be available combinationally for the current `vga_mem_address`. If you use synchronous BRAM with 1-cycle read latency, you will need to pipeline the pixel path.: VGA timing, text-cell addressing, and color selection
- `vga_font_rom.luc` : ASCII glyph ROM (8×8 bitmap font)

## Notes

- The visible-area bounds have been fixed in your version, so the renderer now uses the full **640×480** visible region.
- This module expects `vga_mem_data` to be available combinationally for the current `vga_mem_address`. If you use synchronous BRAM with 1-cycle read latency, you will need to pipeline the pixel path.
