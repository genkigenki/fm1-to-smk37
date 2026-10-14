# 06: Mapping patterns

The SMK-37 has more controls than the FM-1: ten more keys, 16 RGB pads, 4 faders, 2 wheels and 4 more
buttons. These patterns put them to work. Each one ran on a real unit in at least one port. Pick per control
group, write the choice into [`smk37-port.yaml`](../smk37-port.yaml), then implement it.

![The SMK-37 panel](../assets/smk37-panel.png)

Pad numbers in this guide are in reading order: pads 1-8 are the top row (hardware pads 0..7), pads 9-16 the
bottom row (hardware pads 8..15). The hardware index is the same for hits and LEDs; see
[01-hardware.md](01-hardware.md), 6.3. The diagrams are drawn by `tools/make_panels.py`.

**The base rules:**

1. **The firmware's own layout first.** Every FM-1 control keeps a home on the SMK, in the firmware's own order
   and grouping and under its own names, so its manual still applies. The extra controls add to that and never
   take something away. A new firmware (no FM-1 manual to keep) can drop this rule.
2. **No doubles.** A control never repeats another control's job, unless the owner asks for it. Before giving a
   new control a job, ask what has no direct control yet.

## Quick chooser

| Control | Pattern | Used in | Choose it when |
|---|---|---|---|
| Keys | [Middle 27 = FM-1 keys](#keys-the-fm-1-range-in-the-middle) | all ports | always |
| Keys | [Velocity setting](#keys-velocity-setting) | Snoop37 | the firmware has a settings menu |
| Outer keys | [Extend, drums on a drum track](#outer-keys-extend-and-drums) | SLOOP | the firmware has a drum track or a range limit |
| Pads | [Function keys on the pads](#pads-function-keys-on-the-pads) | ChoralRoot | the firmware uses some keybed keys as functions or modifiers |
| Pads | [Steps in reading order](#pads-steps-in-reading-order) | SLOOP, X0X, Felucca PR | the firmware has a 16-step sequencer |
| Pads | [Sound picker](#pads-sound-picker) | SLOOP | many presets, chosen by scrolling on the FM-1 |
| Pads | [Voices and a CHAOS pad](#pads-voices-and-a-big-gesture) | Makkuro37 | a drone or multi-voice instrument |
| Pads | [Strum plate](#pads-strum-plate) | FoMni | a chord instrument |
| Faders | [Banks with soft takeover](#faders-banks-with-soft-takeover) | SLOOP, Makkuro37, FoMni | per-track settings, more than four jobs |
| Faders | [What has no direct control](#faders-what-has-no-direct-control) | | the knobs already reach every parameter on screen |
| Wheels | [Bend and vibrato, or a macro](#wheels-bend-and-vibrato-or-a-macro) | all ports | always |
| Wheels | [Bend range setting](#wheels-bend-range-setting) | Snoop37 | the firmware has a settings menu |
| Wheels | [Calibration in the power-on calibration](#wheels-calibration-in-the-power-on-calibration) | Snoop37 | the firmware has a hardware calibration |
| TRS jack | [MIDI out on the TRS jack](#trs-jack-midi-out) | Snoop37 | the firmware sends MIDI over USB |
| Knobs | [The FM-1 top row](#knobs-the-fm-1-top-row) | SLOOP, X0X, FoMni | keeping the FM-1 workflow |
| Knobs | [A page per button, eight columns](#knobs-a-page-per-button-eight-columns) | Makkuro37 | a redesign with many parameters per voice |
| Buttons | [Printed meaning, four extras](#buttons-printed-meaning-and-four-extras) | SLOOP, X0X, FoMni | the firmware names its buttons by function |
| Buttons | [By row](#buttons-by-row) | | the firmware groups its buttons by row (one row of verbs, one of modes) |
| Whole port | [A toy for children](#a-toy-for-children) | Rainbow mode | a firmware for kids, or any safe-by-default instrument |

## Keys: the FM-1 range in the middle

The middle 27 keys (keybed 5..31, F3..G5) are the FM-1's note keys, with velocity from the unit's own factory
curves (the `trimdat` record, read only). In layers where the FM-1 uses its keys as tiles (steps, effects, mix),
the same keys do the same thing. Used by every port.

## Keys: velocity setting

The FM-1's keys have no velocity, and many FM-1 firmwares assume a fixed one (SLOOP: 100). On the SMK, offer a
KEY VELOCITY setting in the firmware's own settings menu, SMK builds only:

- **Values:** DYNAMIC (as played, the default), SOFT and HARD (two curves on the played velocity), FIXED 64 / 100
  / 127 (100: as the FM-1).
- **Where:** apply it where the firmware reads a key's or a pad's velocity (one helper), so the notes, their MIDI out
  and anything derived from velocity (drum ghost / hard levels) follow it. The raw velocity stays in the input layer.
- **Store:** in free bits of a settings word the firmware already keeps; 0 = DYNAMIC, so older data loads as before.
  No store version bump, the FM-1 build byte-identical.
- **Used in:** Snoop37 (HOME menu > AUDIO > KEY VELOCITY, on K8).

## Outer keys: extend and drums

![Outer keys](../assets/panel/keys-outer.svg)

The 10 outer keys (C3-E3, G#5-C6) extend the keyboard by default: on a synth track they play notes. On a drum
track they play drums, so you can finger-drum while the middle keys keep their FM-1 job. A black key plays its left
white neighbour.

- **SLOOP:** C3 kick, D3 snare, E3 hi-hat; G#5 cowbell, A5 open hi-hat, B5 crash, C6 ride.
- **Choose:** extend when the firmware plays notes; drums when it has a drum track; transpose or off when the
  middle keys are tiles most of the time.
- **Watch:** an outer-key drum hit lights no pad (the pads show steps).

## Pads: function keys on the pads

When the FM-1 firmware uses some of its keybed keys as functions or modifiers (ChoralRoot's chord block: DIM MIN MAJ SUS,
6 m7 M7 9 and LOCK on F3..C#4), move those functions onto the pads and let the whole keybed play. This beats mirroring
the FM-1's keybed onto the pads: the player gets all 37 keys for notes and the functions in a place of their own.

- **Layout by musical meaning:** one group per row or block, in the firmware's own order (ChoralRoot: the top row's
  left half the types DIM MIN MAJ SUS, the bottom row's the extensions 6 m7 M7 9, then LOCK).
- **The original behaviour:** a pad is the key it replaces: held while it is held, combinations as with the keys, and
  it latches only where the original latches (ChoralRoot's LOCK). No latching or one-per-group rule of your own.
- **Light as the key's light:** bright while held or latched, dim otherwise; a colour per group, the screen's where it
  has them (ChoralRoot: types white, extensions orange, LOCK blue).
- **Pads left over: the firmware's mode pickers**, grouped by colour (ChoralRoot: the performance modes, cyan for the
  free-timing ones Strum, Strum 2 Octaves, Slop, Harp, green for the tempo-synced Arpeggiate, Arp 2 Octaves, Pattern). A
  tap picks the mode and switches the function on, the playing mode's pad again switches it off, as the firmware's own
  button does; only the mode playing is bright. Use the firmware's own pick and toggle functions so the button, its
  layer and the pads always show one state.
- **Implement:** route the pads to the firmware's key handlers (press / release of the function key), not through the
  note bits: the keybed keys that carried the functions now play notes and keep their bits.
- **Used in:** ChoralRoot. **Choose:** the firmware has modifier keys on its keybed.

## Pads: steps in reading order

![Pads as steps](../assets/panel/pads-steps.svg)

The 16 pads are the FM-1's 16 white keys F3..G5, in **reading order**: steps 1-8 on the top row, 9-16 on the
bottom (hardware pads 0..7 and 8..15). Convert in one place, for hits and LEDs alike. Pads light in the
screen's track colours: steps that are on are lit, the playhead and hits bright, off is dark (no glow layers).
Pads play with velocity.

- **Used in:** SLOOP (also a piano layout in the SEL layer, save slots in the SAVE layer), X0X (the step
  grid), the Felucca PR.
- **Choose:** the firmware sequences 16 steps, or its white keys are tiles. This is the default.

## Pads: sound picker

![Pad sound picker](../assets/panel/pads-picker.svg)

Turning the sound knob (K4, PRESETS) shows the sounds of one kind on the pads and the screen. A pad picks and
previews, `<` `>` page through the kinds, and a button (KNOB BANK) pins the picker open. After a knob turn it stays
for a short time, then the pads go back to steps.

- **Used in:** SLOOP.
- **Choose:** the firmware has many presets that the FM-1 can only scroll through.

## Pads: voices and a big gesture

![Pads as voices](../assets/panel/pads-perform.svg)

The pads gate voices: the top row wakes and sleeps voice 1..8, the voice's colour on the pad, its knob and the
screen. One pad does a big gesture (CHAOS) that is hard to do with knobs.

- **Used in:** Makkuro37 (eight drone voices).
- **Choose:** a drone, a multi-voice or a live-performance instrument without a step sequencer.

## Pads: strum plate

![Pads as a strum plate](../assets/panel/pads-strum.svg)

The pads are 16 strings: top row low, bottom row high. The keybed holds the chord; a finger across a row strums
it. A string lights while it rings; at rest the root strings glow dimly. Velocity sets loudness and brightness.

- **Used in:** FoMni (the Omnichord-style chord harp).
- **Choose:** a chord instrument, an arpeggiator or anything played by sweeping.

## Faders: banks with soft takeover

![Fader banks](../assets/panel/faders-banks.svg)

Fader i sets track i, in the track's colour. FADER BANK picks what they set: **A** level, **B** filter
(low-pass left, open centre, high-pass right), **C** reverb send, **D** delay send. A fader takes over only once it
moves, also after a bank change, so nothing jumps. A FADER BANK screen shows all four banks; its LED is lit off
bank A.

- **Used in:** SLOOP (as above), Makkuro37 (banks FX L, FX R, LFO, PLAY, OUTPUT), FoMni (banks A and B).
- **Choose:** per-track settings, or more than four things worth a fader.

## Faders: what has no direct control

When banks or a mixer would repeat what the knobs already do, give the faders the amounts that have no control of
their own: a standing effect level that otherwise sits in a menu, a send, a balance between two engines. List the
firmware's parameters, cross out what a knob or button already reaches, and offer the owner the rest.

- **Choose:** the owner's rule is "no doubles" and the knobs cover the screen's parameters.

## Wheels: bend and vibrato, or a macro

![Wheels](../assets/panel/wheels.svg)

Pitch wheel = bend (SLOOP: +-7 semitones on the selected track, Snoop37 2.6.4: a setting, default +-7; FoMni: +-2). Mod wheel = vibrato (SLOOP:
10 Hz, up to +-1 semitone, up = more; FoMni: 5.5 Hz, up to +-1/2 semitone), or one performance macro. The
wheels calibrate themselves: centre taken at power-on, the range widens as the wheel is seen to move, with a dead
band around the centre.

- **Choose:** bend and vibrato unless the instrument has one gesture worth a wheel.

## Wheels: bend range setting

A fixed bend range never suits everyone. Offer BEND RANGE in the settings menu, SMK builds only: +-1, 2, 3, 4, 5,
7, 12 semitones, default the range the port had (Snoop37: 7, so an update keeps the feel). It sets the wheel on the
internal sounds only; a MIDI IN pitch bend range the firmware already has stays separate. The bend code takes the
range as a parameter (1/4096 semitone x range), so the range math is host-testable. Store it in a spare byte of a
settings part the firmware already writes, 0 = the default, so older data loads as before and the FM-1 build is
unchanged. Used in: Snoop37 (HOME menu > MIDI > BEND RANGE, K6).

## Wheels: calibration in the power-on calibration

The wheels' travel differs per unit (pitch ~445..640, mod ~474..629 raw on one unit), so self-calibration only
reaches the full range once the wheel has been at its ends. Add the wheels to the firmware's hardware calibration,
after the buttons and knobs:

- **Steps:** pitch at rest, full down, full up; mod at the bottom, full up. One button (OCT+ as just taught) takes
  each step; the screen shows the live reading. At an end, keep the furthest reading of the whole step (a wheel let
  go a moment early still reached its end); at rest, sample 0.3 s after the press and set the dead band from the
  noise (2 x spread + 6, 8..24 counts).
- **Use:** move the taught ends in by 3 counts so the full travel gives the full range; widen them if the wheel goes
  further. At power-on a pitch wheel resting within the dead band of the taught centre becomes the centre; a held
  wheel uses the taught centre. Keep the slow drift-follow of the centre (in time, not in calls).
- **Store:** a small object of its own (a magic, 16 bytes), not inside a settings struct whose layout other tools
  read; none or out of range = self-calibration as before. Snoop37: `OBJ_WCAL` at 0xB1000 / 0xB2000.
- **Safety:** a 30 s idle timeout per step keeps the old values; too little travel is refused with a message.

## TRS jack: MIDI out

The SMK's 3.5 mm jack is a MIDI OUT (UART1 TX on PC7); the FM-1's is MIDI IN on PH8 (a key row on the SMK: keep
that driver off). Send the same stream as USB MIDI out: hook the one function that queues USB MIDI packets and hand
each packet to the TRS driver too, before the "USB configured" check, so the jack works without a computer.

- **Bytes:** convert each USB-MIDI packet by its CIN (8..E channel messages, C/D two bytes, F single realtime
  bytes, 2/3 system common); drop SysEx and any packet whose status does not match its CIN (a track with MIDI OFF
  can produce status 0xFF). No running status needed.
- **Ring and pace:** a 256-byte ring of whole messages (a full ring drops the message, never half). Drain it from
  the 10 kHz timer tick, nested in the audio render too: one byte when the last has gone (TPND), never a wait loop.
  That gives ~2500 bytes/s against the line's 3125.
- **No hang:** a byte not done in 30 ticks (3 ms) is given up; after 4 in a row the jack goes quiet until the setting
  is set again. A setting TRS MIDI OUT ON/OFF (default ON) in the settings menu.
- **Adapter:** the SMK-37 is listed as type A (a community spec sheet, not measured); tell users to try type B if a
  synth does not respond.
- **Used in:** Snoop37 2.6.4 (`hal/smk_uart_tx.h`, `src/smk_midi_out.h`), not yet confirmed on a unit.

## Knobs: the FM-1 top row

![Knobs as the FM-1 top row](../assets/panel/knobs.svg)

K1-K8 are the FM-1's knobs: K1 is a virtual MASTER pot (the FM-1's pot is absolute; K1 is an encoder, so it
steps a stored value, 24 of 1024 per detent in SLOOP), K2-K4 the encoders, K5-K8 the screen's four dials in
their colours. One encoder step per detent. Keep the order the firmware's screen uses: SLOOP shows SELECT,
ALGORITHM, PRESETS (K2, K3, K4); a firmware that shows SELECT, PRESETS, ALGORITHM keeps that order.

- **Used in:** SLOOP, X0X, FoMni.
- **Choose:** keeping the FM-1 workflow. Matching knob caps make it obvious.

## Knobs: a page per button, eight columns

![A page per button](../assets/panel/knobs-page.svg)

One logic for every menu: a button opens its page, K1-K8 set the page's eight columns (one per voice, or eight
parameters), `<` `>` turn the pages. A value with two choices can sit on the column's pads. The screen draws the
eight columns in the knobs' order, so knob i always means column i.

- **Used in:** Makkuro37.
- **Choose:** a redesign with many parameters per voice; one rule replaces many menus.

## Buttons: printed meaning and four extras

![Buttons](../assets/panel/buttons.svg)

The FM-1's 14 buttons keep their jobs on the SMK button that means the same: PATCH = HOME, CHORD = ENV,
NOTE REPEAT = LFO, SCALE = SEL, PARA = EDIT, GLOBE = GLO, BT = SAVE, SEQ PLAY = PLAY, SEQ REC = REC, OCTAVE-/+ =
OCT-/+. The four extras: KNOB BANK and FADER BANK for banks or pins, `<` `>` for pages (never tempo). Full table:
[04-controls-mapping.md](04-controls-mapping.md). Use the firmware's own button names in its docs (SLOOP calls
SEL what Felucca calls SCL).

## Buttons: by row

The FM-1 has two rows of six buttons; the SMK's grid is two rows of six too. When a firmware groups by row (for
example one row of actions, one of modes), keep its rows: FM-1 top row to SMK top row, bottom to bottom, left to
right. The printed names then no longer match, so the screen and the controls table say what each button does.

- **Choose:** the firmware's manual explains its buttons by row rather than by name.

## A toy for children

A firmware a child plays alone. Used in Rainbow mode (with the full Felucca behind a combo).

- **Volume cap.** The grown-ups set a cap; K1 never goes above it. K1 is a virtual pot, so clamp the pot value
  (`smk_master`) as well as the level, or turning down does nothing for many detents.
- **Fixed velocity.** The layer reads velocity in `key_vel` and in the outer-key queue. Put one switch there
  (Rainbow mode: `SMK_KEY_VELOCITY 0` = every key and pad at 100) instead of editing each place.
- **No menu from the toy.** No stray press opens a menu (not KNOB BANK either). The grown-up controls (a held
  button, a long combo) are hidden on purpose: the owner's exception to "no hidden modes", noted in the setup file.
- **Two UIs.** The toy and the full firmware each get their own mapping (`modes:` in the setup file).
- **Teaching with lights.** The SMK has no key LEDs. Pads as white keys can light the next white key; black keys
  have no pad. Options: the piano layout on the pads (black keys in the top row), or a keyboard strip on the screen.

## Screen and UX rules

These are the owner's rules from the ports so far. Keep them unless the owner says otherwise.

1. **Group by meaning.** Screens are laid out by function, the knob block at the bottom of the screen, not in
   the hardware's order.
2. **Touch, then see.** The screen shows what you touch first: the value next to the thing it changes, and a
   short hint line saying what the control does.
3. **Same colour, same place.** A colour means the same thing on the screen, the pads and the knob caps; the
   screen's layout follows the panel's. A firmware whose screen maps the FM-1's panel (for example its knobs as a
   2x2 block) gets a map of the SMK's panel instead: eight knobs in one row, in the same order.
4. **No hidden modes.** Every state is visible (an LED, a header, a colour). A held button only lends something
   while it is held.
5. **Play first.** Sound on power-on, few menus, nothing that needs the manual to make a noise.
