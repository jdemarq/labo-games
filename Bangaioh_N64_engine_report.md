# Bangaioh (N64) engine — static recompilation, Dreamcast and DS content in the N64 engine

Technical notes on running the Nintendo 64 game *Bakuretsu Muteki Bangaioh* natively on PC and
Android through static recompilation, then using that engine to play the stages of the two later
versions: the Dreamcast port (*Bangai-O*) and the Nintendo DS sequel (*Bangai-O Spirits*).
The document covers methods, data formats, engine addresses and the problems met on each
platform.

## 1. The games and their rights holders

| | N64 | Dreamcast | Nintendo DS |
|---|---|---|---|
| Title | *Bakuretsu Muteki Bangaioh* | *Bangai-O* | *Bangai-O Spirits* |
| Year | 1999 | 2000 (PAL tested, EN/FR/DE) | 2008 (European version tested) |
| CPU | VR4300 (MIPS) | SH-4 | ARM9 + ARM7 |
| Developer | Treasure | Treasure | Treasure |

The games, their code, graphics, music, text and level data remain the property of their rights
holders.

This document contains **no** game data: no ROM or disc bytes, graphics, text, dialogue, level
layouts or audio. It describes methods, file formats and engine addresses only. Anyone
reproducing the work must use their own legally obtained copies, convert them locally, and never
commit game data to a repository.

## 2. Approach

1. **Static recompilation of the N64 game.** The MIPS code is translated to C with N64Recomp and
   built against N64ModernRuntime (system calls, audio, input) and RT64 (RDP/RSP graphics on
   Vulkan, Direct3D 12 or Metal). The result is a native executable that needs the user's ROM at
   first launch.
2. **Hooks instead of rewrites.** Changes to the game are short C functions inserted before chosen
   instructions of chosen recompiled functions (`func`, `before_vram`, code). The original game
   logic is left intact; the hooks redirect file loads, patch tables in RDRAM, and correct a few
   behaviours.
3. **Converting the other versions to N64 formats.** The Dreamcast disc files and the DS ROM are
   converted once, on the user's machine, into editable PNG and text files. At run time, the
   engine rebuilds N64-format files from them and serves them through the hooked file loaders.
   RDRAM above 4 MiB (unused by the game, Expansion Pak size) holds the virtual files.

## 3. N64 ROM identification

| ROM | SHA-1 |
|---|---|
| Japanese original | `2dbfe78f97b8d6e1a33b73d244be831d18b0491e` |
| Japanese + English fan translation (v2) | `805c7f339cf3bd9445c90240cac6b6ace69179cb` |
| Code region only (ROM `0x1000`–`0x914B0`) | `2e103df83c8b2b30acb5d82d0dcafebbec8e5665` |

The code-region hash is the useful one for other translations: a ROM whose code region matches
but whose full hash differs is almost certainly a text or graphics hack and runs correctly. The
launcher accepts such ROMs after a warning.

Pitfall: the runtime's hash check failed on one Windows toolchain even with the right file. The
check was replaced by an explicit SHA-1 comparison done by the game layer.

## 4. N64 engine reference

### 4.1 Files and compression

- Files are stored in a ROM file table (start/end offsets). Most are compressed with **LZYF1000**
  (big-endian header). The Dreamcast uses the same algorithm.
- **Sprite banks (BMF):** header `count, 0x20, off_frames, off_pal, pal_size, off_pix, pix_size`;
  `frames = (off_pal − off_frames) / 8`; each frame is `(u16 w, u16 h, u32 offset)`. A table of
  frame ids (1-based) maps animation frame numbers to frames. Pixels are CI8 (or 4 bpp for effect
  banks); a CI8 frame must have a width that is a multiple of 8 and fit in 2 KiB of TMEM. Frames
  are drawn centred on the object.
- **Bank 2** holds the ordinary enemies, **bank 3** the player robot (frames 0–8: torso, 9 aim
  directions; 9–26: legs), **bank 5** the houses and scenery objects.
- **Level maps** (files 255–301): `u16 w, h, 256, ntiles`, then a 256-colour palette (RGBA5551),
  a u16 type per tile, the u16 cell array (tile index + bits `0x4000`/`0x8000` = horizontal and
  vertical mirror), and 8×8 CI8 tiles. When a map is loaded, the loader adds the file address to
  the header offsets, so after loading they are pointers, not offsets.
- Backgrounds are 320×240 CI8 images drawn with S2DEX background copies.

### 4.2 Important addresses

| Address | Content |
|---|---|
| `0x80096A08` | map list |
| `0x80096AF0` | stage records, 20 bytes each: u16 map, u16 time, u8 music (?), u8 dialogue id, 6 bytes, background pointer (+12), pattern-layer pointer (+16) |
| `0x800961A0` | dialogue records, 16 bytes: script, messages, face bank 1, face bank 2 |
| `0x80098530` | stage-name table, indexed by map number |
| `0x80147DEA` | current stage (s16, 1-based) |
| `0x801C8CD0` | object slot pointer array (slot 7 = stage intro object, slot 14 = player) |
| object `+0x00` | class pointer |
| object `+0x44`, `+0x48` | position (s32, integer part in the high 16 bits) |
| `0x801CEF41` | main state machine byte (state 20 = gameplay scene, 24 = results) |
| `0x80271934` | stage result byte (2 = cleared) |
| `0x8027193C` | stage goal byte (0 once the stage objective is met) |
| `0x802715EF` | "boss destroyed" flag; the clear sequence waits for it |

Gameplay scene (`func_800248A4`): state 6 waits for the intro object; state 8 plays (no player →
failed; goal byte 0 → cleared); states 13–17 run the clear sequence, which waits for the boss
flag above before fading out.

### 4.3 Map cell types (object spawner `func_80025B84`)

| Type | Object |
|---|---|
| 0 | empty |
| 2 | wall |
| 3–6 | slopes (3 ◣, 4 ◢, 5 ◤, 6 ◥) |
| 7–0x10 | breakable blocks and walls (first pass) |
| 0x18–0x19 | crates |
| 0x1A–0x1D | dome robots (affected by gravity) |
| 0x1E, 0x38, 0x39 | item boxes (0x39 = health) |
| 0x20–0x27 | wall guns, 2×2 cells; even types on a floor (vertical mirror = ceiling), odd types on a side wall (horizontal mirror = right wall) |
| 0x30–0x33 | turrets (second pass; deleted if not resting on a breakable support) |
| 0x40–0x41 | walking robots (blue, yellow) |
| 0x44–0x45 | flying soldiers (red, black) |
| 0x5B | generator |
| 0x78 | core |
| 0x80 | player start, 2×2 |
| 0x90–0x91 | triggers |
| 0xC0–0xC4 | houses (fruit and points when destroyed; collapse without support) |

**Wall guns have no sprite.** The spawner creates the gun object and then clears its four cells.
Only three of the N64 maps draw the guns inside their tiles; everywhere else the guns are
invisible. Their object carries a draw flag (`0x0200` in the word at `+0x38`) that keeps it from
being drawn, and forcing animation frames on it shows nothing.

### 4.4 Controls of the N64 game

Found by pressing each input in turn during play:

| N64 input | Action |
|---|---|
| D-pad | move |
| Analog stick | aim and fire (any direction) |
| C buttons | fire in 4 directions |
| B | fire forward |
| A | fly up |
| L | switch character (Riki / Mami) |
| R, Z | bomb (hold to charge, release) |

The stick does not move the robot, which surprises players: default mappings must route
movement to the N64 D-pad.

### 4.5 Busy waits

The game polls in tight loops (vertical sync, music handle). In a recompiled build they burn CPU
and can starve other threads; hooks inside these loops yield to the runtime (wait up to 1 ms for
an external event).

### 4.6 RT64 S2DEX

Bangaioh relies on two S2DEX microcode commands that RT64 did not fully implement: object
rectangles (`G_OBJ_RECTANGLE`) and background copies with scrolling, wrapping and TMEM strips
(`G_BG_COPY`). Both were implemented; without them the backgrounds and stage banners are
garbled.

## 5. Dreamcast version (*Bangai-O*)

### 5.1 Same engine, same formats

- Same LZYF1000 compression; the same containers with **little-endian** integers instead of
  big-endian.
- Sprite banks have the same layout; the N64 uses CI8 with palettes, the Dreamcast direct
  ARGB1555 or 4/8-bit sheets (PTY).
- Level maps `P_MAPnnn.MAP` correspond to N64 files 255–301 with the same pixel format: only the
  16/32-bit fields need byte swapping. About 29 maps are close variants of the N64 ones (88–100 %
  of cells agree, three are identical); about 16 are different levels.
- 141 sprite banks have an exact N64 equivalent (portraits, icons, speech bubbles, effects);
  dialogue scripts use the same format.
- The three languages differ only in images containing text.

### 5.2 Formats specific to the Dreamcast

- **BGY** (one per level): an extra graphics layer as large as the map, about 2 500 8×8 tiles.
  The 16-bit word at offset 18 selects the pixel format: **1 = ARGB1555, otherwise ARGB4444**.
  Reading every file as 4444 garbles about half the levels.
- **PTY**: backgrounds made of pieces, and sprite sheets.
- One 72 016-byte file per level, probably demo input recordings (not confirmed).
- The 44-stage table of the Dreamcast executable (map, time, dialogue) matches the N64 for all 44
  stages. The dialogue table is sorted alphabetically.

### 5.3 Dreamcast content in the N64 engine

- **Level look:** each stage becomes a picture of the whole map (BGY layer + map tiles) and a cell
  picture (type + mirror flags). At run time the engine cuts the picture into deduplicated 8×8
  tiles (key = pixels + type), quantises it to 255 colours, and writes an N64 map file.
- **Scenery objects** (houses, buildings, bridges): on the Dreamcast, tile types 197–284 spawn
  data-driven objects. A symbolic evaluation of each handler of the executable gives the sprite
  sheet, intact and damaged sprites, hit points and collision box. Objects without hit points are
  drawn into the level picture; the others become N64 houses (type 0xC0) created at the end of
  the spawner, with the Dreamcast collision box, hit points divided by 50, sprites appended to
  bank 5 (colours placed in free palette entries), animation substituted, and their support
  checked on all sides.
- **Player robot:** the Dreamcast robot sprites (51 sprites, hip-anchored) replace bank 3 frames,
  keeping Dreamcast size. Other palettes of the bank (variants) take, for each new colour, the
  variant of the N64 colour of closest luminosity. Ordinary enemies are identical on both
  versions.
- **Not converted:** the Dreamcast player shots are missile sprites, the N64 ones are laser lines.
- **Import:** a zip of the disc, the language folder, or any single file of it (the others are
  found next to it).

## 6. Nintendo DS version (*Bangai-O Spirits*)

### 6.1 Data

- NitroFS file system; ARM9 code uncompressed, loaded at `0x02000000`.
- **Stage list:** 184 pointers (European ROM: `0x02080F74`) to stage files. Menu modes:
  tutorial 0–16, *Treasure's Best* 17–43 (27), *Puzzle Stages* 44–61 (18), *Other Stages* 62–183
  (122, ending with a second copy of the tutorials).
- **Stage header:** u8 version (3 = compressed map, 2 = 3 bits per cell), u8 background index
  (table of 8-byte entries), u8 width (×256 px), u8 height (×256 px, 0 = 192), u8 wall style,
  u8 music, u16 object count, u32 object offset, u32 map offset, 16-byte name (character set:
  space `0x01`, digits from `0x02`, `A` = `0x0C`, `a` = `0x26`).
- **Objects (8 bytes):** u16 type (1–56), s16 x, s16 y, u16 flags. Bits 0–2: direction (0 up,
  2 right, 4 down, 6 left); **bit 3 (`0x08`): target**; `0x40`/`0x80`: team. Positions are the
  bottom centre, or the point of contact with the wall for cannons.
- **Map:** one byte per 8×8 cell: 0 empty, 1 slope (orientation from the walls around it),
  2 wall, 3–6 breakable blocks.
- **Compression** (blocks, rounded to 4 bytes): `1ddddd ll` short copy, `01 dist10 len4`,
  `001 dist4 len9`, `000 n5` literals, 0 = end of block.
- **Wall tiles:** chosen from the 8 neighbours (order left, down, right, up, up-left, down-left,
  down-right, up-right; bit set = neighbour not solid) through a 256-byte table; slopes are
  tiles 24–27.

### 6.2 End of a stage

On the DS a stage does **not** end when every enemy is destroyed: it ends when every object
flagged as a **target** (flag `0x08`) is destroyed, even if other enemies remain. Every stage has
at least one target. In the N64 engine, a hook on the goal check of the gameplay scene counts
the converted target objects, forces the goal byte while targets remain, and sets the "boss
destroyed" flag when the last one goes, so the normal clear sequence, results screen and next
stage follow.

### 6.3 Conversion rules

- Cells: walls → 2, slopes → 3–6, breakable blocks → 8–10.
- The DS edges are walls, the N64 needs real ones: every map gets a 2-cell wall border (the
  player can start on the very edge of a DS map, which hid the player). Maps smaller than the
  N64 screen are centred in walls (42×32 cells at least).
- Objects (first object wins a cell): player → 0x80; cannons → wall guns 0x20/0x21 oriented by the
  direction bits; small enemies → flying soldiers 0x44/0x45 (dome robots fall to the floor);
  crates → 0x19; robots → 0x40/0x41; generators → 0x5B; mines → 0x1C; bosses → 0x41.
- **Cannon placement:** the DS position is on or inside the wall. The 2×2 gun block goes into the
  first free cells in front of it and must be entirely free: the spawner clears all four cells,
  and a block overlapping a wall leaves a hole in it.
- 184 stages do not fit in the 44 stage slots of the N64 engine: they are split into pages of up
  to 44 per mode, each page played as stages 1–44 (dialogue removed, time 600).

## 7. Making wall guns visible (Dreamcast and DS modes)

1. Take the gun drawings from the three N64 maps that contain them (16 pictures: 4 gun kinds × 4
   directions, rotated or mirrored where a direction is missing).
2. Draw them into the level picture before quantisation, with a heavy weight in the colour
   histogram (a few gun pixels are otherwise merged into the scenery colours).
3. Hook the spawner just after each of the four gun creators (they clear the cells and jump to the
   end of the loop) to record the cells and the object slot.
4. At the end of the spawner, put the original gun tiles back and set their type to 0 in the
   map's type table. Putting the cells back during the spawner makes it spawn the same gun again
   at the next cell.
5. Each gameplay frame, clear the cells of the guns whose object is gone.

Pitfall: the renderer only draws correctly the tiles that the map referenced when it was
loaded. Swapping a cell to a tile that appears nowhere else in the map displays some other
tile; restoring the original gun tiles avoids it.

## 8. Platforms

| Platform | State |
|---|---|
| Linux, Windows | N64, Dreamcast and DS modes playable |
| Android (Adreno) | built on an existing Android port of the runtime and renderer |
| Android (Mali-G615, driver r44p1, Vulkan 1.3) | runs, but palette textures are garbled (see 8.2) |

### 8.1 Defaults for a 2D game

Interpolating to the display refresh rate, MSAA and native resolution cost a lot of GPU time
on handhelds for no visible gain on sprites and tiles. Defaults: original frame rate, no MSAA,
and 480p on Android. A profile of the game thread showed it waiting on the renderer, never
computing.

### 8.2 Mali: open problem

Symptoms: images loaded with a palette (backgrounds, banners, large sprites) show bands of
other images and swapped red/blue channels; small map tiles are correct. The same renderer
patches with the same capability gates render correctly on desktop drivers, including a
software Vulkan driver.

Ruled out, one at a time:

- queue family ownership (the driver exposes a single queue family);
- dual-source blending (the device does not advertise `dualSrcBlend`; the fallback path and
  specialised shaders change nothing, but the gate is still needed on Mali);
- non-coherent host memory (flushing and invalidating mapped buffers made it worse);
- CPU-side SIMD vector math on ARM64 (scalar mode changes nothing).

Next steps: compare with an older Mali (G52) to tell a driver-generation bug from a general Mali
one, then capture a frame with RenderDoc (needs a debuggable build) to find the operation the
driver gets wrong.

Notes for Android work: Mali has no user-loadable open driver (the open Vulkan driver needs a
kernel driver the device does not have), unlike Adreno. Logs are hard to reach on recent
Android: showing the renderer messages of the previous run in the launcher was the practical
way to get them.

## 9. Controls

| Action | Keyboard | Gamepad | Touch (Android) |
|---|---|---|---|
| Move | WASD (same keys on AZERTY: ZQSD) | D-pad or left stick | stick |
| Fire, 4 directions | arrows | A/B/X/Y (direction of the button) | diamond |
| Aim and fire, any direction | IJKL | right stick | — |
| Fire forward | Space | right stick click | FIRE |
| Fly up | Shift | left stick click | UP |
| Switch character | Q (A on AZERTY) | L1 / L2 | SWITCH |
| Bomb | E | R1 / R2 | BOMB |

On a handheld whose controls can be covered (screen closed), the touch stick and diamond plus the
four shoulder buttons give the full set.

## 10. Pitfalls met (read this before redoing it)

- Two image formats share the Dreamcast BGY extension (offset 18 selects 1555 or 4444).
- Map header offsets become pointers after loading.
- A CI8 sprite frame wider than TMEM allows (2 KiB) or not a multiple of 8 is drawn wrong.
- Some N64 objects (turrets) are deleted by the spawner if their support is not breakable.
- Hooking the stage intro check instead of the goal check froze the intro.
- The clear sequence waits for the "boss destroyed" flag: forcing the goal byte alone freezes the
  game with the timer stopped.
- The renderer draws correctly only the tiles referenced by the map at load time.
- When restoring cells during the spawner, the spawner meets them again: restore at its end.
- DS cannon positions are inside the wall; DS maps need a wall border in the N64 engine.
- Test on the target GPU early: desktop drivers tolerate things mobile drivers do not.

## 11. Redoing it: checklist

1. Recompile the N64 game; hook the busy waits; implement the S2DEX commands it uses.
2. Identify the ROM by SHA-1 of the whole file and of the code region.
3. Hook the file loaders and move virtual files above 4 MiB of RDRAM.
4. Dreamcast: decompress (LZYF), swap endianness, read BGY with the right pixel format, convert
   maps, backgrounds, dialogue, scenery objects and robot sprites.
5. DS: read NitroFS, the stage list and stage files, decompress maps, convert cells and objects
   (cannon placement, wall border), map targets and hook the goal check.
6. Draw the wall guns and restore their cells at the end of the spawner.
7. Set default controls for the game (movement on the N64 D-pad) on every input device.
8. Validate on each platform, including the GPUs of the target handhelds.
