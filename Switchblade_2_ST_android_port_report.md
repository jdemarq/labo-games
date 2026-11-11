# Switchblade II (Atari ST) — native port status report

Status of the native port of the **Atari ST version** of *Switchblade II* (Gremlin Graphics,
1991) to portable C99, running on Android. It describes the method, the data and memory
layout the port relies on, what has been ported and verified, how verification works, and what
is still open.

Every address is an absolute 68000 address in the running game unless stated otherwise.
Status as of November 2026.

## 1. The game and its rights holders

| Title | **Switchblade II** (Atari ST version, 1991) |
|---|---|
| Publisher | **Gremlin Graphics** |
| Genre | side-scrolling platform action, six levels, shop between zones |

The game data remains the property of its rights holders. The port ships no game data: the
build reads it from the player's own copy of the game.

## 2. Approach

Unlike the first *Switchblade* (machine-generated static recompilation, see the companion
report), *Switchblade II* is ported by **hand transcription** of the 68000 routines into C:

- The C code reads and writes a **512 KB image of the ST's RAM** (`$0`–`$7FFFF`, big-endian
  accessors). Every game variable stays at its original address, so the port's memory can be
  compared byte for byte with an emulator dump.
- Each routine is transcribed instruction by instruction: same branches, same flag semantics
  (signed `bmi`/`bpl` on 8- or 16-bit results, `dbra` counts, word-indexed tables `(a0,d0.w)`),
  same side effects on registers that survive the call (for example `d7` or the upper word of
  `d0` that a callee leaves untouched).
- Rendering, sound output and input are the only parts written for the host; the game logic
  never knows it is not on an ST.
- Nothing the game's own memory carries is hard-coded in the port: tables, per-level
  geometry and constants are read from the RAM image.

## 3. Data

**Archive.** The game's files are gathered into one archive (`sb2.pak`) built from the
original disks: about 100 members (graphics, maps, per-level RAM images), each stored raw or
compressed.

**Compression.** The game's own `*FUNGUS*` codec (backwards LZ) is reimplemented instruction
for instruction; every compressed member decodes byte-exact against a run of the real decoder
(97/97 members of disk 1, 41/41 of disk 2).

**Per-level RAM image.** For each level `N`, a member `ram_n<N>.chm` holds the RAM zones the
engine needs (format: magic `SB2RAM1\0`, zone count, then address, length, bytes). At level
start the port lays out, in the order the game loads them:
1. the zones of `ram_n<N>.chm`;
2. the staged objects listed in the table at `$83EA` (41 entries of 12 bytes: slot pointer,
   size, file name; entries pointing at `$614BA` are not loaded);
3. seven resident blocks, identical in all six levels;
4. the tile atlas at `L($31DC)`, the level grid at `L($32EA)`, the spawn map at `L($41D6)`,
   the spawn-map extension at `L($41DA)`;
5. the block graphics at `$6E828` (40,000 bytes, up to `$78468`).

The zones were measured from emulator dumps taken at the first object pass of each level,
and the result is audited to 0 differing bytes against those dumps in the regions the engine
reads.

## 4. Memory map and key structures

| Address | Content |
|---|---|
| `$439E` | current level, 1–6 |
| `$43A0` | level geometry: 8 bytes per level, **two longs** (rows S, columns C); cell (C, R) of the grid is byte `C*S + R` |
| `$5204` / `$5208` | scroll origin in the grid (recomputed by the camera code) |
| `$520C` / `$520E` | camera x / y |
| `$5210` / `$5212` | hero world x / y |
| `$5C10` / `$5C14` | head / tail of the live object list |
| `$5C18`–`$635B` | object pool 1: 31 slots of `$3C` bytes; free stack `$6360`–`$63D8`, pointer `$635C` |
| `$63DC` / `$63E0` | live list of pool 2 (hero shots and weapon effects) |
| `$63E4`– | object pool 2; free stack `$6C1C`–`$6C94`, pointer `$6C18` |
| `$7054` | behaviour jump table (identical in all levels) |
| `$12B9E` | hero weapon slots (8 × 8 bytes) |
| `$7158` | tile attribute table; `$36BA` points at the attribute under the hero |

**Object slot layout:** `+0` x, `+2` y, `+5` image, `+6` graphics pointer, `+$A`/`+$E`
previous/next, `+$12` pointer to the spawn-map byte, `+$16` behaviour, `+$17` hit points,
`+$18` flash, `+$1A` linked part (multi-part enemies), `+$1E`… behaviour-private data.

**Self-modifying code.** At level start the game rewrites immediates inside its own code
(grid height, half-height, camera limit, exit-tile switch, and more). The port reads those
constants from the RAM image at the patched addresses instead of copying values.

**Main loop, one pass at 25 Hz:** weapons (`$12BEE`) → hero (`$110F0`) → camera
(`$110FC`–`$1137C`) → objects (`$113C2`, `bsr $13AF0`) → draw preparation (`$1D0B2`) →
status, bars, death and end-of-level checks. The host calls the ported functions in this same
order.

## 5. What is ported

| Part | State |
|---|---|
| Archive, codec, screen and sprite formats, level grids, palettes | done, byte-exact |
| Hero: movement, jumps, slopes, collisions, sword, damage, death, respawn | done |
| Camera and object spawning by scroll (column and row scans) | done |
| Object behaviours | **all entries of `$7054`** (0–56; entry 57 is null in the game itself) |
| End-of-level bosses (behaviour 44 dispatching to one routine per level) | done; boss graphics reloaded from the archive |
| Hero weapons: basic gun and the five shop weapons | done |
| Shop: entry, prices, money, purchase matrix, exit back to the level | done |
| End of level, HUD (status bar blitter), game over, high-score table, name entry | done |
| Sound effects (YM2149 driver data) | done |
| Android host | done (see §6) |

**Notes worth knowing:**
- The behaviour routines are byte-identical in the six level dumps (only one immediate
  differs), so any dump can serve as the source.
- The hero **does** have a gun: pressing fire with no enemy in sword range sets a flag that the
  weapon routine turns into a shot from pool 2, with a muzzle flash returned the next pass.
- Level 5's water is a trap, not a swimming mode: below `y = $350` the hero routine simply
  returns. The level is played above the water (crouch-jump over the first pit).

## 6. Android host

- One native library (C99, NDK) plus a thin Java shell: full screen, back key captured,
  gamepad and touch controls, title screen.
- The 320 × 200 frame is presented in an 8:5 surface, centred, fixed orientation; optional
  CRT lines.
- Pause menu: freeze, save states, a cheat toggle, display settings, quit.
- Audio: YM2149 synthesis on the audio thread only (owning the chip on one thread removed the
  crackles), AAudio output.
- Verified on an Android handheld: all six levels started and played through a test hook
  that selects the level at launch.

## 7. Validation

**Traces.** The reference is the real game under the Hatari emulator. A capture stops at the
object pass (`$113C2`) and right after it (`$113C6`) every pass and dumps `$0`–`$9000`.
A converter turns the dumps into a text trace:
- `P k` starts pass k;
- `D addr val` is a byte written by the rest of the game since the previous pass;
- `R` runs the objects pass;
- `E addr val` is a byte the pass itself changed;
- `F addr val` is a byte forced by the capture driver (cheats such as invulnerability).

**Two modes, both must give 0 differing bytes:**
1. **Per pass:** the port's state is resynchronised from the trace before each objects pass.
   This checks every behaviour in isolation.
2. **Chained:** the port runs the whole loop (hero, camera, spawning, objects, status) on its
   own from pass 0; only the joystick comes from the trace. This checks that nothing outside
   the object code drifts. Display buffers and the sound driver's voices are excluded.

A trace is accepted into the test set only at 0 differing bytes in both modes.

**Current set:** 29 traces covering all six levels and the five shop weapons, from 8 to 600
passes each, plus unit benches for the codec, level loading, hero, end of level, shop
purchase, sound, the host glue and a synthetic bench for two behaviours no map ever spawns.
The host test suite passes in full.

## 8. Capturing with Hatari

- **Pick the level during the title screen:** write `$439E` while the title is shown. The
  game sets it to 1 during boot, so writing it earlier is overwritten. Always check `$439E`
  in the first dump.
- **Teleport for coverage:** at pass 0, before the full dump, write the camera
  (`$520C`/`$520E`), the hero's screen position (`$522C`/`$522E`) and `$3180`. The hero falls
  where he is; objects spawn at the next scroll, so land next to the target and walk to it.
  Rehearse the route on the port first: it replays the same inputs and shows when each
  behaviour first appears.
- **Inputs:** write the joystick byte and the fire byte in the script of each pass. Fire as
  pulses (two passes pressed, two released): held fire strikes only once.
- **Cheats:** invulnerability and lives are re-written after every pass and declared to the
  converter as `F` lines.
- **Parallel runs:** with a command FIFO, several emulator instances can run at once, but on a
  2-CPU machine more than four delay boot past the title and the level write is lost.
- Each trace carries, on its second line, the command that reproduces it.

## 9. Pitfalls met (read this before redoing it)

- **Read the level geometry as longs.** Read as words, `$43A0` looks empty.
- **The RAM image does not contain the whole code area.** A routine that reads a table inside
  the code (for example a direction table at `$16A7A`) needs that zone added to the image,
  otherwise it silently reads zeros.
- **Keep game state in game memory.** Caching a value in a host-side field (the slope
  attribute under the hero) broke the chained mode on level 3; storing the pointer at `$36BA`
  as the game does fixed it.
- **Per-level main-loop details:** level 6 increments a colour-cycling counter (`$317C`)
  every pass; missing it shows up only in chained mode.
- **Callee side effects matter.** Some routines leave registers the caller later tests (the
  upper word of `d0`, `d7`); transcribe what the callee really does to them.
- **Breakpoints from boot are slow and may retrigger;** arm the capture breakpoints once the
  level is running.
- **Debugger writes take decimal bytes unless prefixed with `$`.**

## 10. Open items

- The water trap of level 5 (`$1D132` → `rts`) is not mirrored in the hero routine; no trace
  reaches it, since all captures stop at the water line in agreement.
- One shop animation frame is left at a fixed value (an accepted, documented deviation).
- The credits screen is packed in a format not yet identified.
- Further long play sessions on device to look for anything the traces did not exercise.

## 11. Redoing it: checklist

1. Build the archive from the original disks; check every compressed member decodes
   byte-exact (§3).
2. Dump the RAM of each level at its first object pass; derive the per-level image zones and
   audit the rebuilt image against the dump (§3).
3. Transcribe the main loop in the game's order, reading every table and patched constant
   from the image (§2, §4).
4. Port behaviours from the `$7054` table; for each, capture a trace near where it spawns
   (spawn map + teleport) and require 0 bytes in both modes (§7, §8).
5. Add the hero weapons, shop, end of level, HUD and high scores the same way.
6. Write the host last: frame presentation, input mapping, audio thread (§6).
