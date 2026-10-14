# 08: Pitfalls

Things that went wrong in earlier ports, and the fix. Check this list before step 1 and again before a release.

## Flash and data

| Pitfall | Fix |
|---|---|
| A fork moved its flash map (Melodee 1.0's layout crosses the SMK's protected ranges). | Redesign the layout inside the allowed windows (store `0x9D000-0xBAFFF`, samples `0xC2000-0xE9FFF`). Never write `0x0-0x3FFF`, `0xC0000-0xC1FFF`, `0xF4000-0xFDFFF` or anything from `0xFF000`. Assert every block's end address. |
| Two firmwares read each other's flash data after a switch and misbehave. | Each firmware gets its own data family / owner tag (`FELUCCA_DATA_FAMILY` in the reference series). A store with another tag is treated as empty. |
| FM-1 settings at `0xFC000`/`0xFD000` land inside the SMK's protected USR area (`0xF4000-0xFDFFF`). | Move them into the store window. Check every object against the protected ranges, by its `OBJ_*` enum, not only the addresses a grep finds. |
| Two objects share one A/B sector pair (told apart by a record type), so an address list shows one object. | Keep them sharing the pair under their own types, or give each its own pair. Extra objects go into `0xB1000-0xBAFFF`. |
| The Felucca layer's `_Static_assert(OBJ_COUNT == 10u)` stops the build. | Intended: the fork has a new object. Give it a place, then extend the assert and the flash-map test. |
| The owner-tag test fails after the family was changed. | `tests/smk_owner_test.c` may hard-code `SMK37DATAFELUCCA`; take the family from `FELUCCA_DATA_FAMILY` there too. A 7-letter family fills the 16-byte tag with no NUL: compare 16 bytes, not `sizeof` (17). |
| The project's name has 8 letters or more (GHOULBOX); the family takes 1..7. | Shorten it (`GHOULBX`). |
| A fork keeps its own data in "unused" settings bytes that a later upstream release uses (felucca-rainbow's song in `favorites.factory[14..15]`, Felucca 1.2's PRESETS LIST byte). | Share the upstream data family only if the settings layouts match byte for byte; otherwise give the port its own family. |
| Cherry-picking the Felucca layer onto an older fork: the FM-1 half of `fm1_flash_map.h` carries 1.4.1's `FL_VBANK`, and the FM-1 store range grows. | Copy the FM-1 values from the fork's own `fm1_flash.h` into `fm1_flash_map.h`, never the layer's. Check the FM-1 hashes. |
| The first boot clears the stock firmware's settings (untagged data). | Expected with `FELUCCA_DATA_UNTAGGED_OK 0`; calibration and the way back are safe. Tell the owner before the first install. |
| A new setting or calibration needs room, but the settings struct is read at fixed offsets (SLOOP's web editor edits bytes 48..83) and older firmware checks the object's length. | Use free bits / spare bytes (TRS MIDI OUT: settings word bit 31, 1 = OFF; BEND RANGE: a spare byte of the MIDI part, 0 = default), or a small object of its own (the wheels' calibration). Avoid `0xB0000` (the stock firmware's VM data; an untagged store keeps it); extras from `0xB1000`. Erase the extra sectors too when the owner tag changes. |
| An upstream merge reuses a settings bit the port already uses (SLOOP 2.6's MIDI channel vs Snoop37's BOOT LOGO bits). | After every upstream merge, diff the settings struct and its bit masks. Move the port's bits to free space and migrate old values. |

## Identity, models and installers

| Pitfall | Fix |
|---|---|
| The SMK comes as SMK-37 Pro, SMK-37 Elite and MKE-P37; the stock updater only takes a unit's own model. | Build one package per model: `BOARD=smk37 MODEL=elite sh build.sh` (`pro`, `elite` or `mke`). Each reports its model's name, so the stock updater can restore the factory firmware. |
| An older layer (Felucca 1.5.1) has no `MODEL`; `MODEL=elite` is silently ignored and builds a Pro package. | Use the current layer (Felucca 1.6 or SLOOP 2.6), whose per-model commit adds `SMK_MODEL`. |
| A per-model commit taken from another port hard-codes that port's names (Rainbow's family default and `rainbow37` stem). | Replace them in build.py, build.sh, BUILDING.md, run_tests.sh and `tests/smk_owner_test.c` with the project's: `<project>-smk37`. |
| Starrykey 37 Play's name, `STARRYKEY-37 PLAY` plus a version, is 21 characters; a 20-block package identity holds 20. | Use the SMK-37 Pro build on it. Installers accept any SMK model's package on any SMK model (never FM-1 to SMK or back). Read the stock V016 package's identity past 20 blocks (it takes 21, then the unused 0x7D markers) and the device identity in full (bytes 6..32). |
| Ports shipped as renamed products ("xxx37") with the porter's credit on the boot screen read as taking over someone else's work. | The SMK-37 build keeps the project's name, logo and boot; packages `<project>-smk37[-elite\|-mke].fwsc`; the goal is an upstream PR (AGENTS.md, "The goal"). |
| The installer does not find a keyboard running another port. | Port-name filters must include every firmware's USB name. Add yours to the installers you ship. |
| Another firmware's identity reply ends its text early; string compares fail. | Parse identities up to the first NUL byte. |
| Windows shows MIDI ports as `MIDIIN2 (name)` and similar. | Match port names by substring, case-insensitive. |
| The installer's `resume()` (device already in update mode) wrote any loader, while `install()` checked the model: with SMK loaders accepted, an SMK in update mode could get an FM-1 package (found in a SLOOP review; Felucca 1.1.5 forks have it too). | `resume()` takes the package identity and checks the loader's model by `install()`'s rule (equality, or the same-keyboard rule); without the identity it writes nothing. Every caller passes it. Test both: wrong model and no identity, nothing served. Reference: the layer's `web/fm1ota.js` and its tests (AGENTS.md, step 8). |
| The fork changed `resume()` itself (GHOULBOX: a 4th argument `waitMs`, a reload path); the reference file and the fork's clash. | Take the reference file, re-add the fork's parts as options (`opts.waitMs`), convert every caller and test. The test harness's fake device differs per fork too. |

## Build and tests

| Pitfall | Fix |
|---|---|
| Windows clones turn LF into CRLF; patches stop applying and hashes change. | `git clone -c core.autocrlf=false URL` (after `clone`, so it is stored) or `git config core.autocrlf false` and check out again. `git -c core.autocrlf=false clone` does not persist. Check: `git config --local core.autocrlf` prints `false`. |
| `__DATE__` in the firmware makes builds differ from day to day. The JieLi clang ignores `SOURCE_DATE_EPOCH`, and a fixed define would add a flag to the FM-1 build. | Compare on the same day as the baseline, or rebuild the baseline from the base commit on the day of the comparison (`git worktree add ../fm1-base <base>`), or run both builds under `faketime`. |
| The tests fail after the SMK build. | Both boards write the same `build/` intermediates. Build the SMK, copy its package out, build the FM-1, then run the tests. |
| A big test suite fails without naming the failing part. | `grep -n "FAIL\|OVER BUDGET"` the log, or make the runner print each suite's name. |
| Font and image tools fail with `ImageFont has no attribute 'Layout'`. | They need Pillow with raqm; distribution packages are often too old. `python3 -m venv ~/.venv-fw && ~/.venv-fw/bin/pip install Pillow fonttools`; check `features.check('raqm')`. |
| Commands sent from Git Bash to WSL lose their `$` variables. | Run builds and tests inside WSL; put longer commands in a script and run `wsl bash script.sh`. |
| The firmware's own agent file says to merge or push automatically. | The kit's rules win: work on `smk37`, the owner merges and pushes. |
| An upstream PR edited the live site: `docs/` is generated from `web/` by `make_site.py` and published by the maintainer (found in a SLOOP review). | A PR changes sources only. Leave `docs/` and other generated or published directories to the maintainer's build. Put SMK docs at the top level (`SMK37.md`) or elsewhere outside the published tree. |
| A merged port broke the maintainer's browser emulator (`web/emu/build.sh`): the shared sources used `FL_JEDEC_OK` and `FM1_OCT_BOTH` from `hal/`, which the emulator does not include (found in a MAKU merge, reverted). | Build every target the repo has before a PR: the firmware, the emulator / simulator and what CI runs. Define new shared macros where the emulator can see them too (e.g. in `web/emu/*.c` with the FM-1's values). |
| `web/emu/build.sh`: Permission denied in a fresh clone. | Run it with `sh web/emu/build.sh`. |
| The web tests fail on Windows node (doubled drive-letter paths from `new URL(".", import.meta.url).pathname`, a drive colon in `path:key` arguments), or are skipped. | Run them in WSL with node on PATH there (emsdk's: `. ~/emsdk/emsdk_env.sh`). |
| A test file written through a shell heredoc got real NUL and newline bytes where the C source had `\0` and `\n`. | Check generated sources for control bytes before committing (`grep -P '[\x00-\x08]'`). |
| Host tests on glibc fail on `clock_gettime` and friends with `-std=c99`. | `CC='cc -D_DEFAULT_SOURCE' sh tests/run_tests.sh` (some forks need `-D_GNU_SOURCE`, others nothing). |

## Fork code

| Pitfall | Fix |
|---|---|
| The fork has its own boot stub in front of the app (ChoralRoot: `firmware/stub`, a commit marker, docs/BOOT-SAFETY.md). It carries its own copies of the FM-1's register sequences: it drives PA2 low (the FM-1's matrix rail) and bit-banges a 16-bit 595 word. On the SMK PA2 is a matrix row. | Give the stub an SMK variant behind `BOARD_SMK37`: never touch PA2 (no rail on the SMK); shift 32 bits by hand in the SPI DMA's order (the word's bytes little-endian, each msb first); clear the SPI2 pin mapping first (a warm reset keeps the app's); OCTAVE- is column 6, OCTAVE+ column 3, both on row PH8; the staged loader at 0xBB000 and the plain window from 0x9D000. Keep app.bin's slot, the body and the marker at the FM-1's places (the SMK's PRCT is larger, the rest stays 0xFF). The stub's update record names the model: one stub lock per model. Check the boot on the unit: the installer's final identity line, then `--debug`'s stub line (samples, the PH register). |
| A fork stores far more than Felucca (ChoralRoot: 19 objects and 20 loop sectors) and the store window 0x9D000-0xBAFFF has 30 sectors. | A fork without samples can use USRFLASH (0xC2000-0xE9FFF, `FL_SMP`) for its larger objects (banks, loop slots). Keep one list of every data region and use it for the owner change and the firmware's own "erase the data" menu alike; the owner tag stays at 0xAF000 and out of that list. |
| The fork's LED code builds column bitmaps from `FM1_KEYMAP` positions, with the FM-1's ids (buttons 0..13, keys 14 + k). The SMK's button ids go up to 17 and collide with the key ids. | Keep the fork's composition and give the SMK a plain id space (id / 8, id % 8) with the key ids moved up (ChoralRoot: `CU_LK(k)`); one SMK output function at the end reads the levels back for the button LEDs and the pads. |
| A byte-identity break from an "equal" constant: `id < NB` (an int enum) replaced by `FM1_NBTN` (`14u`) changes the comparison's signedness and can change the code. | Wrap the old expression in a macro that expands to exactly the old text on the FM-1 (`#define PANEL_NBTN NB`) and to the SMK's value under `BOARD_SMK37`. Check the FM-1 hashes after each area. |
| The fork's installers classify the running firmware (ChoralRoot: `classify_firmware` / `classifyFirmware`): every identity that is not `FM-1_...` is refused as unknown, so the SMK port cannot be installed over the SMK's own firmware or another port. | Add the SMK rows to both tables (the stock `SMK-37 Pro_0NN` etc., the ports' 7xx / 9xx, the fork's own INFO text; `SMK-37_9NN` refused as `other`), keep the two tables equal (the fork's own Python-vs-JS test), and use the SMK installer behaviour on every path (model rule, our loader only, long identities). |
| The fork's host tools need a newer Python than the WSL default (ChoralRoot's `gen_tables.py` uses `math.exp2`, Python 3.11). | A venv from `uv` with Python 3.12 and Pillow / fontTools; compare base and new in the same environment. |
| A fork's own mode replaces `ui_input()`, `ui_leds()` and `ui_draw()` (felucca-rainbow's toy); the pads, faders, wheels, outer keys and extra buttons do nothing there. | Grep every reader of `fm1_in`, `panel_enc`, `fm1_input_edges` and every writer of `fm1_led*`; wire the SMK extras into each mode that owns the frame. |
| A loop `for (id = 0; id < 14u; id++)` over the FM-1 buttons: on the SMK, FX, SEQ, PLAY and REC (ids 14..17) are dead. | Use `FM1_NBTN` (14 on the FM-1, 18 on the SMK). Grep forks for `14u`, `27u` and `14u + k`. |
| A fork's LED code with `uint8_t nl[FM1_NCOL]` and key LED ids `14u + k` writes out of bounds on the SMK (`FM1_NCOL` is 16, LED ids go up to 63). | Write an SMK version of that LED function (button LEDs and pads), not just a recompile. |
| The fork added its own power-on combos and flash checks with FM-1 values (GHOULBOX: USB RESCUE as `(buttons & 3u) == 1u`, JEDEC `== 0x856014u`); the layer's `FM1_OCT_BOTH` / `FL_JEDEC_OK` cover only Felucca's places. | Grep for `buttons & 3u`, `& 1u) ==`, `0x856014` and fix each (OCTAVE- is button 4 on the SMK). |
| Fork code reads an `fm1_in` field the SMK input lacks (GHOULBOX's crank: `enc_dfr`), or counts in FM-1 frames (909/s; the SMK's are 625/s). | Add the field to `smk_input.h`; grep for frame-based timing constants. |
| The fork refactored `key_on` (GHOULBOX: `key_on_to(t, note, chord, chn)`, no key index); the layer's `key_vel(k)` in `key_on(k)` no longer fits. | The caller passes the velocity in. |
| Outer keys go straight to `input_on` / `input_off`: no chord, latch or split from the fork. | Route them through the fork's key_on / key_off helpers (own chord buffers, 10 slots); add them to features that list the held keys. |
| On-screen "K1".."K4" in an FM-1 firmware means KNOB 1..4 (Snoop37's HOME menu rows); on the SMK those knobs are K5..K8 (K1..K4 are MASTER, SELECT, ALGORITHM, PRESETS). | Fix every such label behind `BOARD_SMK37` (menu rows, hints, help lines, `SMK37.md`); grep UI strings and `'1' + i` style labels. The hardware calibration's "KNOB 1".."KNOB 4" too (Snoop37: K2 SELECT, K3 ALGORITHM, K4 PRESETS, K5..K8). The FM-1 build stays byte-identical. |
| A volume cap applied to the master level after `master_poll` leaves K1's virtual pot running past the cap; turning down then does nothing for many detents. | Clamp the virtual pot (`smk_master`) to the cap's pot value too (in Felucca master = pot^2 / 256). |

## Hardware

| Pitfall | Fix |
|---|---|
| Newer WS2812 parts need a longer reset than the stock firmware sends; a glitch shifts the chain and a pad lights its neighbour. | Send a long reset (about 250 bytes of zeros) before each pad LED frame. |
| Docs, comments and console texts disagreed on which row is which. | Verified on a unit: top row = hardware pads 0..7, bottom = 8..15, the same index for hits and LEDs ([01-hardware.md](01-hardware.md), 6.3). Convert in one place; check with `padtest` and `smk` at bring-up. |
| Two-colour button LEDs: one LED index is the other half of a neighbour (SEQ PLAY's red half). | Take LED indices from the reference series, confirmed on a unit. |
| The SMK gives no VBUS on its USB-C port. | USB host experiments need a powered hub or an external 5 V supply. |
| Faders and wheels are noisy near the ends. | A dead band of +-1 step, ends reachable; wheels self-calibrate from the power-on centre. |
| The FM-1's TRS MIDI IN driver (PH8) switched on for the SMK. | PH8 is a key-matrix row: keep `FELUCCA_UART` 0 and the `#error`; the SMK's jack is MIDI OUT on PC7, its own driver. |
| PC7 driven as plain GPIO for MIDI OUT: nothing comes out. | Set it up as the stock firmware does: `IOMAP_CON3` bit 7 clear and [23:20] = 1 (UT1 TX onto an output channel), PC7 PU = 1, PD = 1, output low, DIE = 1; UART1 `CON0` 0x3400 then UTEN, `BAUD` 383 on PLL48M. Send: set `CON0` bit 13 (clear TPND), write `BUF` (0x1210C, a byte); bit 15 (TPND) when gone. PC7 is the FM-1's LCD chip select: an SMK-only setup. |
| A blocking `putbyte` (wait for TPND) in the audio or UI path. | Poll from the timer tick, one byte per tick, with a timeout; a missing UART must never stall anything. |
| The update loader includes `usb.c` too: a TRS hook in `midi_out_event` breaks the loader build (implicit declaration). | Guard the hook with `!defined(FELUCCA_LOADER)`. |
