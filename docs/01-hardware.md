# 01: SMK-37 hardware map (compared with the FM-1)

This is the datasheet the port needs. The SMK-37 Pro, SMK-37 Elite and MKE-P37 (and the Donner Starrykey 37 Play)
share this hardware and run the same firmware (one package per model name). Each section lists the SMK-37 facts,
how they differ from the FM-1, and what a port must do.

Status tags:

- **verified**: checked on a real SMK-37 Pro with a custom firmware (the reference port's debug console, a
  flash read over the mask-ROM bootloader, or plain use).
- **analysis**: read from the stock firmware (version 016) with concrete constants, but not yet confirmed on
  hardware.
- **open**: not known yet.

Register addresses are physical SFR addresses of the AC791N. Pin names follow JieLi's port naming (PA0..PH15).
Where the stock firmware uses numeric pin ids, PA0 = 0x00, PB0 = 0x10, PC0 = 0x20, ..., PH0 = 0x70.

---

## 1. SoC, boot and memory

| Item | SMK-37 | FM-1 | Status |
|---|---|---|---|
| SoC | JieLi AC791N (WL82 family), pi32v2 core | same | analysis |
| External flash | 1 MiB SPI NOR, JEDEC id **0xEB6014** on the tested unit | 1 MiB, 0x856014 | **verified** |
| Second-stage bootloader (SPL) and boot config | byte-identical to the FM-1 (the whole flash range 0x0000-0x3FFF matches) | | analysis |
| Chip key | 0x980F | 0x980F | analysis |
| App image execution address | **0x02000120** (XIP); app.bin starts at flash 0x4120 | same | **verified** |
| Update loader (`ota.bin`) | byte-identical to the FM-1's | | analysis |
| Clock tree, reset config | identical boot config, so the same crystal and PLL setup | | analysis |

The flash part differs from the FM-1's. Firmware that allow-lists the JEDEC id (Felucca and SLOOP refuse to
write to an unknown part) must add **0xEB6014**. The driver only uses
generic SPI NOR commands (`9F`, `05`/`35`, `06`, `20`, `02`, `0B`), so adding the id is enough. Until you do,
the firmware runs without saving and the update loader refuses to write, which is safe. Other units may carry
a different part: read the id during the flash backup (see
[02-update-and-recovery.md](02-update-and-recovery.md)).

## 2. Flash map

The 1 MiB map, as defined by the region table of the stock v016 package (status: analysis):

| Flash range | Stock SMK-37 Pro v016 | FM-1 (stock v15) |
|---|---|---|
| `0x00000-0x03FFF` | header, SPL, boot config | same, identical bytes |
| `0x04000-0x9CFFF` | app area (PRCT): app head at 0x4000, `app.bin` at 0x4120, then config blobs | `0x04000-0x92FFF` |
| `0x9D000-0xBFFFF` | VM: key-value settings store | `0x93000-0xE8FFF` |
| `0xC0000-0xC0FFF` | BTIF: Bluetooth pairing | `0xE9000` |
| `0xC1000-0xC1FFF` | **USRTRIM: factory calibration** (SMK only) | none |
| `0xC2000-0xEAFFF` | USRFLASH: user data, probably presets (SMK only) | none |
| `0xEB000-0xF3FFF` | not listed in the region table | |
| `0xF4000-0xFDFFF` | USR: stock resources | `0xEA000-0xFBFFF` |
| `0xFE000-0xFEFFF` | not listed | (SLOOP keeps settings here on the FM-1) |
| `0xFF000-0xFFFFF` | key_mac | same |

Two details trip up existing tools:

- The region table entries for VM, BTIF, USRTRIM, USRFLASH and USR hold **absolute** flash offsets. They are
  not relative to the app area at 0x4000. Some unpackers add 0x4000 and report the regions 16 KiB too high.
- Region flags as stored in the stock table: VM `12 81`, PRCT `92 82`, BTIF `92 82`, USRTRIM `92 81`,
  USRFLASH `92 81`, USR `92 81` (index 1). A package built for the SMK should carry the same table, so that the
  official loader treats it the same way.

### 2.1 The calibration record (USRTRIM, 0xC1000)

| Offset | Size | Contents | Stock default |
|---|---|---|---|
| +0x000 | 8 | header (not set by the defaults path) | |
| +0x008 | 16 x 5 x u16 | pad curves: 5 rising points each | 50, 550, 720, 850, 950 |
| +0x0A8 | 37 x 5 x u16 | key velocity curves: 5 rising points each | 5, 640, 800, 920, 980 |
| +0x21A | 5 x u16 | one extra curve (used for the pedal) | 20, 850, 920, 980, 1010 |
| +0x224 | 8 | marker `trimdat\0` | |

The stock firmware accepts the record only if the marker matches and the curves rise strictly. Otherwise it
writes the defaults back to flash. A port should **read** this record (it holds the per-key tuning of your unit),
check it the same way, and fall back to the defaults in RAM, **without ever writing the sector**. A quick self
test at boot is to read 8 bytes at flash 0xC1224 and expect `trimdat\0`. Status: layout from analysis. The
reference port reads the record this way, and key velocity with the factory curves works on the tested unit.

### 2.2 Data map used by the reference port

| Range | Use |
|---|---|
| `0x04000-0x9CFFF` | app area. Only the update loader writes here. App slot (XIP length) `0x97400`, the size of the stock app. |
| `0x9D000-0xBAFFF` | the firmware's own store: projects, presets, autosave, settings |
| `0xBB000-0xBFFFF` | update staging (the custom loader and its update record) |
| `0xC2000-0xE9FFF` | user sample slots (2 x 80 KiB). This overwrites the stock USRFLASH data. |
| everything else | not written |

The update loader may only write `0x04000-0x9CFFF` (app) and `0x9D000-0xBFFFF` (its records, which must stay in
VM so that cleaning them up can never reach BTIF or USRTRIM).

## 3. Display

| Item | SMK-37 | FM-1 | Status |
|---|---|---|---|
| Controller | ST7789V, IPS panel | ST7789-class | **verified** |
| Resolution | 240 x 240, no window offset | 240 x 240 (FM-1 init sets a row offset) | **verified** |
| Bus | **SPI1 group A** (`IOMAP_CON1` bit 4 = 0): **CLK PB6, MOSI PB7** | SPI1 group B (bit 4 = 1): CLK PC9, MOSI PC10 | **verified** |
| Chip select | **PC8**, active low | PC7 | **verified** |
| Data/command | **PB5** (low = command) | PC8 | **verified** |
| Reset | none (the panel relies on power-on reset) | | analysis |
| Backlight | no control found. It is on whenever the unit is on. | PA2, active low | verified (on without any pin driven) |
| SPI1 registers | `0x11D00` CON, `+4` BAUD, `+8` BUF, `+0xC` DMA address, `+0x10` DMA count | same | |
| SPI1 setup | CON `0x4021`, BAUD 4 (about 12 MHz) | same | **verified** |
| Pixel format | RGB565, high byte first, MADCTL 0x00, COLMOD 0x05 (16 bpp), inversion on (INVON) | same idea | **verified** |

The stock panel setup, as register writes (the port uses the same sequence):

| Step | Command | Data |
|---|---|---|
| 1 | wait 100 ms after SPI setup | |
| 2 | `11` SLPOUT, then wait 120 ms | |
| 3 | `36` MADCTL | `00` |
| 4 | `3A` COLMOD | `05` |
| 5 | `B2` PORCTRL | `0C 0C 00 33 33` |
| 6 | `B7` GCTRL | `35` |
| 7 | `BB` VCOMS | `32` |
| 8 | `C2` VDVVRHEN | `01` |
| 9 | `C3` VRHS | `15` |
| 10 | `C4` VDVS | `20` |
| 11 | `C6` FRCTRL2 | `0F` |
| 12 | `D0` PWCTRL1 | `A4 A1` |
| 13 | `E0` positive gamma | `D0 08 0E 09 09 05 31 33 48 17 14 15 31 34` |
| 14 | `E1` negative gamma | same 14 bytes |
| 15 | `21` INVON | |
| 16 | clear the full 240 x 240 window to black, then `29` DISPON | |

An FM-1 port keeps its LCD driver code (command, data, window, fill). Only the pins, the SPI1 group bit and the
init table change. An ST7789 minimal init also lights the panel, but the table above is what the panel was tuned
with.

## 4. RGB pad LEDs

| Item | SMK-37 | Status |
|---|---|---|
| LEDs | 16 WS2812-class RGB LEDs under the pads, one data line | **verified** |
| Data pin | **PC10** = SPI1 MOSI in **group B** (`IOMAP_CON1` bit 4 = 1). PC9 (group B clock) is left alone and stays an analog input. | **verified** |
| Encoding | one SPI byte per LED bit, MSB first: `0xF0` = 1, `0xC0` = 0 | **verified** |
| Bit clock | SPI1 BAUD 8 (about 6.67 MHz, 1.2 us per LED bit) | **verified** |
| Frame | 36 zero bytes (reset) followed by 16 x 24 bytes: 420 bytes, about 0.5 ms by DMA | **verified** |
| Colour order | **GRB** | **verified** |

Chain order (hardware pad index to position in the chain):

| Pad | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Chain slot | 0 | 15 | 14 | 13 | 12 | 11 | 10 | 9 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |

The LEDs and the display share SPI1, on two different pin groups. The stock firmware squeezes the LED frame into
a gap in its LCD transfer. A port can do it more simply:

1. Wait until the LCD transfer has finished and the LCD chip select (PC8) is high.
2. Set `IOMAP_CON1` bit 4 and BAUD 8, then DMA the 420-byte frame from RAM.
3. Wait for the DMA to finish, then clear bit 4 and restore the LCD baud rate.

Never switch the pin group while a DMA is running. Flush the LEDs from the main loop, for example once per UI
frame when a colour has changed. Keep PC10 as a low output between frames, so the latch time is always met.

## 5. Key matrix: keys, buttons, encoders

### 5.1 Shift registers and return lines

| Signal | SMK-37 | FM-1 | Status |
|---|---|---|---|
| Column drivers | **4 x 74HC595** (32 bits), shifted by **SPI2 group C**: CLK PA3, MOSI PA4 (`IOMAP_CON1` bits 17..16 = `10`) | 2 x 74HC595, bit-banged on PA3/PA4 | **verified** |
| Latch | **PA1**, pulse high after each shift | PA1 | **verified** |
| Output enable | PA0, held low | | analysis |
| Return lines (rows 0..7) | **PA2, PA5, PA6, PA7, PA9, PH6, PH8, PH9**, inputs with pull-ups, low = contact closed | PA0, PA5..PA8, PB7 | **verified** |
| SPI2 registers | `0x11E00` CON, `+4` BAUD, `+0xC` DMA address, `+0x10` DMA count. Setup: CON `0x4020` (master, transmit), BAUD 5, then enable bit 0 | | **verified** |

PH6, PH8, PH9 and PA9 are LED or MIDI pins on the FM-1. On the SMK-37 they are matrix inputs. **Driving them
as FM-1 outputs shorts them against the matrix. Remove the FM-1 LED and MIDI-IN code paths for these pins.**

The 32-bit word goes out as 4 bytes by DMA (little-endian in memory):

| Bits | Use |
|---|---|
| 31..24 | column 0..7 select, active low, one-hot |
| 23..16 | column 8..15 select, active low, one-hot |
| 15..0 | button LED lines (active low), **except bits 5..3**, which carry the analog mux address |

So the column word for column `c` is `~(0x01000000 << c)` for c < 8 and `~(0x00010000 << (c - 8))` for c >= 8.
The LED word and the mux address are combined into the low half with AND masks.

The matrix has **16 columns x 8 rows**. Rows 0..4 carry the keybed, rows 5..6 the buttons and row 7 the
encoders. A position is written below as a code `col << 4 | row` (hex: high digit = column, low digit = row).

**Scan order matters.** Each step: read the rows (they belong to the column latched one step earlier), pulse
the latch for the word shifted during the last step, then start the DMA of the next column's word. With this
order, the codes in the tables below apply unchanged. With a different order they shift by one column. The
reference port runs one step per tick of a 10 kHz timer, so a full frame (16 columns) takes 1.6 ms. The stock
firmware runs the scan from the SPI2 interrupt and is faster.

### 5.2 Keybed: 37 keys with two contacts each

Each key has a first contact (closes early in the stroke) and a second contact (closes near the bottom).
Key `k = 0` is the lowest key (C3), `k = 36` the highest (C6) (verified in use).

| k | c1 | c2 | k | c1 | c2 | k | c1 | c2 | k | c1 | c2 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 0 | 10 | 52 | 10 | c0 | 62 | 20 | 61 | 83 | 30 | 64 | 24 |
| 1 | 20 | 72 | 11 | d0 | 42 | 21 | 71 | 93 | 31 | 74 | 34 |
| 2 | 30 | 92 | 12 | e0 | 22 | 22 | 81 | a3 | 32 | 84 | 44 |
| 3 | 40 | b2 | 13 | f0 | 12 | 23 | 91 | b3 | 33 | 94 | 54 |
| 4 | 50 | d2 | 14 | b0 | 32 | 24 | a1 | c3 | 34 | a4 | d4 |
| 5 | 60 | f2 | 15 | 11 | 33 | 25 | b1 | d3 | 35 | b4 | f4 |
| 6 | 70 | e2 | 16 | 21 | 43 | 26 | c1 | e3 | 36 | c4 | e4 |
| 7 | 80 | c2 | 17 | 31 | 53 | 27 | d1 | 23 | | | |
| 8 | 90 | a2 | 18 | 41 | 63 | 28 | e1 | 13 | | | |
| 9 | a0 | 82 | 19 | 51 | 73 | 29 | f1 | 14 | | | |

**Velocity.** The stock algorithm, which the port reuses:

1. When contact 1 closes, note the time.
2. When contact 2 closes (debounced), compute `T = 10 + elapsed time in 0.1 ms units`. If both contacts close in
   the same scan, T = 10.
3. `x = 1023 - T` (or 1 if T > 1022). Faster strokes give larger x.
4. Look x up in the key's 5-point curve `t[0..4]` from the calibration record:
   - x <= t[0]: velocity 1
   - x > t[4]: velocity 1023
   - otherwise, for the segment i where `t[i-1] < x <= t[i]`:
     `v = (i - 1) * 256 + (x - t[i-1]) * 256 / (t[i] - t[i-1])`
5. Send note-on with MIDI velocity `max(1, v >> 3)`.
6. Note-off when contact 1 opens (after a few stable scans). Releasing only contact 2 keeps the note on.

With the default curve, T up to about 4 ms gives full velocity. Strokes slower than about 100 ms give the
minimum. The port's 1.6 ms frame resolves fast strokes coarsely. If hard playing all reads as maximum, scan the
key rows more often.

### 5.3 Buttons: 18 positions on rows 5 and 6

| id | Code | Printed name | id | Code | Printed name |
|---|---|---|---|---|---|
| 0 | 26 | KNOB BANK | 9 | 45 | CHORD |
| 1 | 76 | FADER BANK | 10 | 15 | GLOBE |
| 2 | 96 | < | 11 | 86 | BT |
| 3 | 56 | > | 12 | 95 | PATCH |
| 4 | 66 | OCTAVE- | 13 | 55 | PARA |
| 5 | 36 | OCTAVE+ | 14 | 85 | FX |
| 6 | 75 | ARP | 15 | 35 | SEQ |
| 7 | 65 | NOTE REPEAT | 16 | 16 | SEQ PLAY |
| 8 | 25 | SCALE | 17 | 46 | SEQ REC |

Status: **verified** (all 18 names checked on a unit). Debounce of 2 to 4 scan frames works well.

### 5.4 Encoders K1..K8: quadrature on row 7

| Encoder | A | B |
|---|---|---|
| K1 | 97 | a7 |
| K2 | 37 | 47 |
| K3 | 17 | 27 |
| K4 | b7 | c7 |
| K5 | 57 | 67 |
| K6 | 77 | 87 |
| K7 | d7 | e7 |
| K8 | f7 | 07 |

Status: **verified** (K1..K8 in this order).

Decoder (stock method): build the 2-bit state `A | B << 1` each frame. Accept a new state only when it has
been stable for two frames. Keep the last three states in a 6-bit history. Count +1 or -1 only when the last two
transitions went the same way:

- one direction: history `0x07, 0x1E, 0x21, 0x38`
- other direction: history `0x0B, 0x12, 0x2D, 0x34`

This yields 4 counts per quadrature cycle. **One detent is a whole cycle**, so divide by 4 for one UI step per
click. (The stock divisor of 2 gives two steps per click.) Use the reference port's sign convention. If a knob
runs backwards on your unit, flip the sign.

### 5.5 Button LEDs: 19 single-colour LEDs on two commons

| LED | = button | Group, bit | LED | = button | Group, bit |
|---|---|---|---|---|---|
| 0 | KNOB BANK | 0, 12 | 10 | GLOBE | 0, 11 |
| 1 | FADER BANK | 1, 2 | 11 | BT | 0, 10 |
| 2 | < | 0, 0 | 12 | PATCH | 1, 1 |
| 3 | > | 1, 0 | 13 | PARA | 1, 9 |
| 4 | OCTAVE- | 0, 13 | 14 | FX | 1, 8 |
| 5 | OCTAVE+ | 1, 13 | 15 | SEQ | 1, 10 |
| 6 | ARP | 0, 2 | 16 | SEQ PLAY | 1, 12 |
| 7 | NOTE REPEAT | 0, 1 | 17 | SEQ REC | 1, 14 |
| 8 | SCALE | 0, 9 | 18 | (spare) | 1, 11 |
| 9 | CHORD | 0, 8 | | | |

- The LED lines are the low 16 bits of the 595 word ("bit" above), **active low**.
- The two group commons sit on the second USB PHY's pads used as GPIO: register `0x16A0C`, **bit 17 = group 0,
  bit 18 = group 1**, low = group on. When writing the register, keep its GPIO-mode bits set (`| 0x1000F`).
- Multiplex the groups: one scan frame shows group 0 with its LED bits, the next frame group 1. Switch the
  common at the start of a frame (column 0).

Status: **verified** for the LEDs the reference port drives, analysis for the rest of the table.

## 6. Analog inputs

### 6.1 ADC and muxes

| Item | SMK-37 | FM-1 | Status |
|---|---|---|---|
| ADC channels | **ch1 = PA8** (mux A), **ch8 = PC9** (mux B), **ch2 = PA10** (mux C) | ch3 PB1 (battery), ch4 PB6 (MASTER pot) | **verified** |
| Muxes | 3 x 8:1, all three share one 3-bit address = **bits 5..3 of the 595 word** | none | **verified** |
| Pin mode | PA8, PA10, PC9: analog (input buffer off, no pulls) | | **verified** |
| ADC registers | `0x13100` CON, `0x13104` result (10 bit) | same | |
| ADC timing | **CON = `0x400D | ch << 8`, then set bit 4 (enable), then bit 6 (start)**. Done flag: bit 7. A conversion takes about 40 us. | FM-1: `0xF04E`/`0xF06E`, polled | **verified** |

**Use the SMK timing.** The FM-1's ADC setting (long start delay, slow divider) takes about 200 us per
conversion. At one conversion per 100 us tick it never finishes, and the pads and faders read nothing. This
was one of the bring-up bugs.

The mux address you set is shifted out with the next 595 word, so it takes effect one step later. The reference
port cycles through 8 addresses x 3 channels (24 steps, 2.4 ms) and converts one channel per tick. Before each
conversion it also clears bit 14 of register `0x11900`, as the stock firmware does.

### 6.2 What sits on each mux address

| Address | Mux A (ch1, PA8) | Mux B (ch8, PC9) | Mux C (ch2, PA10) |
|---|---|---|---|
| 0 | pad 1 | pad 9 | fader index 1 |
| 1 | pad 2 | pad 10 | fader index 2 |
| 2 | pad 3 | pad 11 | fader index 3 |
| 3 | pad 0 | pad 8 | fader index 0 |
| 4 | pad 4 | pad 12 | **pitch wheel** |
| 5 | pad 7 | pad 15 | **mod wheel** |
| 6 | pad 5 | pad 13 | pedal jack |
| 7 | pad 6 | pad 14 | battery |

### 6.3 Pads

16 force-sensitive pads with velocity and pressure.

- **Physical layout: the top row is hardware pads 0..7 (mux A), the bottom row is pads 8..15 (mux B)**, both
  left to right. Status: **verified** with `padtest` on a unit (2026-10-10). Earlier notes, console texts and
  comments in some layers say the opposite; they are wrong.
- One number for each pad: the hardware pad index is the same for the analog reading (the muxes) and for the LED
  (through the chain-slot table in section 4). Count in this index everywhere and convert to reading order in one
  place (the HAL's `SMK_PAD_OF_WHITE` / `SMK_WHITE_OF_PAD`).

| Position | top 1 | top 2 | top 3 | top 4 | top 5 | top 6 | top 7 | top 8 | bottom 1 | bottom 2 | bottom 3 | bottom 4 | bottom 5 | bottom 6 | bottom 7 | bottom 8 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Reading order (kit) | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 |
| Hardware pad (ADC and LED) | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 |
| Mux | A | A | A | A | A | A | A | A | B | B | B | B | B | B | B | B |
| LED chain slot | 0 | 15 | 14 | 13 | 12 | 11 | 10 | 9 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
- Each pad has a slowly tracking baseline (start at 100; step down by 1 when the reading is below it; step up by
  2 while below the pad curve's first point).
- The reference port: a hit starts at baseline + 24. The peak of the next 3 samples goes through the pad's
  5-point curve from the calibration record (same lookup as the keys) to give the velocity. Release below
  baseline + 12 for two samples. Thresholds are tunable by feel.
- Pressure while held (stock formula): raw <= 512 gives `raw >> 4`; above that `min(127, (raw - 512) / 4 + 32)`.

### 6.4 Faders

- Four faders on mux C addresses 0..3 (index mapping in the table above). The reference port uses index 0..3 as
  FADER 1..4 (verified in use).
- Full scale reads **1023** (verified).
- Filter: a median of 3 against single-sample spikes, then a slow IIR, then 0..127 with a +-1 dead band that
  still reaches both ends. Use a **signed** filter. An unsigned IIR wraps when the fader moves down.

### 6.5 Wheels

| Wheel | Mux C address | Raw range on the tested unit | Status |
|---|---|---|---|
| Pitch | 4 | rest about **535**, travel about **445..640** | **verified** |
| Mod | 5 | about **474..629**, **up = lower raw value** | **verified** |

The travel is short and differs between units. The reference port calibrates itself: it takes the pitch centre at
power-on, lets the centre follow slow drift while the wheel rests, and widens each range to the extremes it has
seen. It uses a dead band of about 12 counts around the centre. Snoop37 2.6.4 also teaches the wheels in the
hardware calibration (pitch: rest, full down, full up; mod: bottom, full up), for the full range on the full travel.

### 6.6 Pedal and battery

- **Pedal**: mux C address 6. It reads about 985 with nothing plugged in. The stock firmware inverts it and maps it
  through the extra 5-point curve in the calibration record. No plug-detect pin was found. Status: analysis.
- **Battery**: mux C address 7. The stock level thresholds are raw 530, 560 and 602 (4 levels). The divider ratio
  is open.

## 7. Power

| Item | SMK-37 | Status |
|---|---|---|
| Power switch | a hard slide switch. There is **no soft power latch**, so the firmware does not need to drive any pin to stay on. | analysis |
| Charger status | **PB1** input, low = charging. | analysis |
| Reset pin | PB1 held low for 8 s resets the chip (boot config, same as the FM-1) | analysis |
| Low-battery "off" | the stock firmware stops the scan, blanks the pads and shows LOW BATTERY. It is a quiet state, not a power-off. | analysis |

**Never drive PB1 and never enable its pull-down.** Do not copy the FM-1's ADC init, which uses PB1 as a battery
channel.

## 8. Audio

The audio path (DAC, output stage, sample rate setup) is the same as on the FM-1. The reference port needed no
change to the audio driver, and the synth engines run unchanged. Status: **verified**.

## 9. MIDI

| Item | SMK-37 | FM-1 | Status |
|---|---|---|---|
| TRS jack | **MIDI OUT** | MIDI IN | analysis |
| UART | UART1 (`0x12100`), **transmit only** | UART1, receive on PH8 | analysis |
| Pin | **PC7** | PH8 | analysis |
| Baud | 31250, from the 48 MHz UART clock (BAUD register 383) | same formula | analysis |
| USB MIDI | class-compliant, as on the FM-1 | | **verified** |

**Disable the FM-1's TRS MIDI IN driver.** PH8 is a key-matrix row on the SMK-37. The reference port forces
the UART option off for this board and fails the build if it is switched on.

MIDI OUT on PC7 (stock setup, from the v016 init): `IOMAP_CON3` bit 7 clear, bits [23:20] = 1 (UT1 TX onto an output
channel); PC7 pull-up and pull-down set, output low, input enabled; UART1 `CON0` = 0x3400 (pendings cleared), the
baud, then UTEN, no RX. A byte: set `CON0` bit 13 (clear TPND), write `BUF` (`0x1210C`); `CON0` bit 15 (TPND) is set
when it has gone. Snoop37 2.6.4 has a driver built this way (status: analysis, not yet confirmed on a unit). The
jack is listed as **type A** in a community spec sheet; M-Vave's manual only says "a 3.5 mm to MIDI adapter" (not
measured).

## 10. USB

| Item | SMK-37 Pro | FM-1 |
|---|---|---|
| Stock VID:PID (analysis) | 4353:B64B | 4C4A:C755 |
| Stock strings | "SMK-37 Pro Midi", "SMK-37 Pro Audio" and others | "FM-1 Midi", "FM-1 Audio" |
| Stock update loader | 4D4A:4155, name from the update record | same |
| Mask-ROM bootloader | `WL82 UBOOT1.00`, 4C4A:8057, USB mass storage (**verified**) | same |

Felucca-family firmware brings its own USB descriptors (VID/PID, audio, MIDI, serial). They work unchanged on the
SMK-37 (**verified**). Host tools that look for the device by port name must also accept names containing
`SMK`.

## 11. Identity and update

| Item | SMK-37 Pro | FM-1 |
|---|---|---|
| Stock identity (SysEx and package) | `SMK-37 Pro_016` (14 characters) | `FM-1_015` |
| Identity of a custom build | `SMK-37 Pro_9XY` (reference port) | `FM-1_9XY` |
| Recovery identity | `SMK-37 Pro_000` | `FM-1_000` |
| Name in the update record | `ota-SMK-37 Pro_016` (18 bytes, zero padded) | `ota-FM-1_015` |
| Package container | **36 interleave blocks** (stock v016) | 20 blocks |

Details are in [02-update-and-recovery.md](02-update-and-recovery.md).

## 12. Summary: what a port changes

| Area | FM-1 code | SMK-37 change |
|---|---|---|
| Board switch | none | one compile-time flag (`BOARD_SMK37`) so that the FM-1 build stays byte-identical |
| Input | 2 x 595 bit-banged, 11 x 6 matrix, 27 keys, 7 encoders, 4 LED lines | new input HAL: SPI2 DMA, 16 x 8 matrix, 37 velocity keys, 18 buttons, 8 encoders, 19 LEDs, muxed ADC with pads, faders, wheels |
| ADC | polled MASTER pot and battery | through the muxes, stock SMK timing. MASTER becomes a virtual pot on an encoder. |
| Display | SPI1 group B, CS PC7, D/C PC8, backlight PA2 | SPI1 group A, CS PC8, D/C PB5, no backlight pin, SMK init table |
| Key LEDs | 27 key LEDs | 16 RGB pads on SPI1 group B, shared with the LCD |
| Flash | FM-1 map, JEDEC 0x856014 | SMK map with protected ranges, JEDEC 0xEB6014 added |
| Update | identity `FM-1_9xx`, 20-block packages | identity `SMK-37 Pro_9xx`, update record name `ota-SMK-37 Pro_016`, 36-block stock packages |
| MIDI | TRS IN on PH8 | off (PH8 is a matrix row). TRS OUT on PC7 (UART1 TX; Snoop37 2.6.4, unconfirmed on a unit). |
| Audio, USB, engines | | unchanged |
