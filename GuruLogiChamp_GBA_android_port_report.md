# Guru Logi Champ (GBA) — Android port: state of the reconstruction

Review of a private project repository that attempts to port the Game Boy Advance puzzle game
*Guru Logi Champ* to a native Android application. This report describes what the repository
really contains, what works, what does not, and what a genuine port would require.

The figures below were checked against the files themselves, not taken from the project README.

## 1. The game and its rights holders

| Title | **Guru Logi Champ** (ぐるロジチャンプ), sometimes written *Guru Logic Champ* |
|---|---|
| Platform | Game Boy Advance, Japan only |
| Year | 2001 |
| Developer / publisher | **Compile** |
| Genre | Puzzle: build a target picture on a block board by firing blocks in and pulling them out, the board rotating so you can act from any side |

The game, its code, graphics, music, text and level data remain the property of their rights
holders. The Game Boy Advance header and boot logo belong to Nintendo.

### Copyright policy for this document

This report contains **no** game data: no ROM bytes, no graphics, no text from the game, no level
layouts and no audio. It describes only structures, methods and the state of the project's own
code. Anyone reproducing the work must supply their own legally obtained copy of the cartridge.

## 2. What the repository contains

```
README.md
android-project/
  build.gradle, settings.gradle, gradle.properties
  gradle/wrapper/ (jar + properties), gradlew.bat      # no Unix gradlew script
  app/build.gradle, proguard-rules.pro
  app/src/main/AndroidManifest.xml
  app/src/main/java/com/gurulogic/game/MainActivity.java
  app/src/main/java/com/gurulogic/game/GameRenderer.java
  app/src/main/res/ (layout, launcher icons, strings, themes)
  app/src/main/assets/
    10 × .bin files  (64 KiB to 2 MiB each, ~11 MiB total)
    level_metadata.json   (22 records)
    level_tiles.json      (4,761 records, ~0.9 MB)
    level_entities.json   (82,598 records, ~15 MB)
```

The `extracted/` folder and the Python extraction scripts mentioned in the README are **not** in
the repository; the README paths point to a local Windows folder on the author's machine.
Some README characters were also lost (`ndroid-project`, `ssets/`, `iles.bin`), typical of a
PowerShell backtick escape (`` `a ``, `` `t ``) eating the first letter.

## 3. Findings

### 3.1 The `.bin` "assets" are not extracted assets

All ten files start with the GBA cartridge header (entry branch + Nintendo logo). Comparing them:

| Files | Size | Content |
|---|---|---|
| `tiles`, `maps`, `objects`, `scripts`, `dialogues`, `text_data`, `audio_samples` | 1 MiB | **byte-identical** |
| `sprites`, `music` | 2 MiB | **byte-identical** |
| `palettes` | 64 KiB | — |

Every file is a **prefix of the same ROM dump**: the 1 MiB files equal the first 1 MiB of the
2 MiB files, and `palettes.bin` equals their first 64 KiB. The names (`tiles`, `music`…) are
labels only; no file was cut at a real data boundary.

### 3.2 The "JSON trilogy" is the ROM sliced into 12-byte records

Each JSON record has the shape `{index, type, coordX, coordY, flags, data}`:

- `index` is a byte offset in the ROM, always a multiple of 12;
- the 12 bytes at that offset are split mechanically: 1 byte → `type`, 2 → `coordX`,
  2 → `coordY`, 2 → `flags`, 5 → `data`.

Checked on the first record of each file: concatenating the fields gives back exactly the ROM
bytes at `index`. The records of `level_metadata.json` start at offset 0, i.e. the cartridge
header itself. Those of `level_entities.json` decode as **ARM machine code** (data-processing and
stack instructions), not as game objects. `level_tiles.json` and `level_entities.json` select
different offsets by some filter, but neither holds tile, map or entity structures.

So no game format was decoded: the "coordinates", "flags" and "types" have no meaning.

### 3.3 The Android code does not compile and does not use the data

`GameRenderer.java`:
- uses **single quotes for strings** (`'Guru Logic Champ'`, `'level_tiles.json'`): in Java these
  are character literals, so the file does not compile;
- `loadGameData()` is never called; it reads `level_tiles.json` into a buffer that is then
  discarded; there is no `org.json` parsing despite the README;
- the 240×160 bitmap (GBA resolution) is created empty and drawn unscaled at the top-left;
- the render thread spins without frame pacing and busy-waits when the surface is not valid;
- there is no input handling, game logic, palette or tile decoding.

The other Java files and resources use single quotes too (fine in XML, not in Java).

Build configuration:
- `classpath 'com.android.tools.build:gradle:4.2.2'` is incompatible with `namespace` and
  `compileSdk 34` in `app/build.gradle` (both need AGP 7.x/8.x);
- the root `build.gradle` uses `allprojects { repositories }` while `settings.gradle` sets
  `FAIL_ON_PROJECT_REPOS`, which makes the build fail;
- no Unix `gradlew` script; Java 8 target;
- the manifest asks for `INTERNET` (unused) and forces **portrait**, while the game is landscape.

### 3.4 README claims vs reality

| README item | Actual state |
|---|---|
| ROM binary parsing — completed | ROM copied in fixed-size prefixes, no parsing |
| Extraction to JSON — completed | Fixed 12-byte slicing, no semantic fields |
| Android asset deployment — completed | Files are in `assets/`, but they are the ROM |
| Java JSON engine — completed | Does not compile; JSON not parsed; nothing rendered except a title |
| Entity logic & collision — open | Correct: not started |

## 4. Legal point to fix first

The project repository currently stores ~2 MiB of the commercial ROM (several times over) in the
assets and in the JSON files. Even in a private repository, this is a copy of copyrighted code
and data. Recommended:

1. remove `app/src/main/assets/*.bin` and the three JSON files;
2. **purge them from Git history** (`git filter-repo --path app/src/main/assets --invert-paths`
   or BFG), then force-push;
3. add `*.gba`, `*.bin`, and the generated asset folder to `.gitignore`;
4. ship tools that extract data **at runtime or at build time from the user's own ROM**, as
   done for Switchblade, never the data itself.

## 5. What a real port needs

Two viable approaches, as for *Switchblade*:

**A. Static recompilation of the game code** (closest to the Switchblade method)
- CPU: ARM7TDMI, mixed **ARM and Thumb** code; ROM mapped at `0x08000000`, IWRAM
  `0x03000000`, EWRAM `0x02000000`, I/O `0x04000000`, palette RAM `0x05000000`, VRAM
  `0x06000000`, OAM `0x07000000`.
- Trace code from the entry point and interrupt handler, separating ARM/Thumb by `BX` targets;
  watch for code copied to IWRAM at boot and for jump tables.
- Translate to C, then build for Android with the NDK.
- Runtime to emulate: PPU (tile modes 0–2, possibly bitmap modes, sprites/OAM, affine
  backgrounds — the rotating board likely uses them), DMA, timers, interrupts, keypad, Direct
  Sound + the 4 PSG channels, and the save type (to identify).

**B. Re-implementation from decoded data**
- Find the real formats: palettes (15-bit BGR, 16/256 colours), 4bpp/8bpp tiles, tilemaps,
  sprites, puzzle definitions, text encoding (Shift-JIS or custom), and any compression
  (GBA BIOS LZ77 `0x10`, Huffman, RLE are common — check for `SWI 0x11/0x12` calls).
- Locate them by following pointers from the code, or by watching VRAM/palette writes in a
  debugger emulator (mGBA, No$GBA) while a puzzle loads.
- Rewrite the rules (block firing, pulling, board rotation, win check) natively and load
  levels from the user's ROM at first launch.

Approach A gives an exact copy faster; approach B gives cleaner code but requires decoding every
format and rewriting the rules.

## 6. Checklist to restart properly

1. Clean the repository (section 4).
2. Write a ROM loader that verifies the user's file (size, header, checksum) without
   distributing it.
3. Disassemble with Ghidra (ARM/Thumb, GBA loader) and map the main loop, VBlank handler and
   level-loading routine.
4. Pick approach A or B; validate each step against an emulator (frame-by-frame screenshots and
   RAM comparisons, as done for Switchblade).
5. Fix the Android project: AGP 8.x, a valid repository setup, a Unix `gradlew`, landscape,
   integer scaling of the 240×160 frame, touch controls mapped to the GBA buttons.
6. Update the README to describe the real state.

## Sources

- [Guru Logi Champ — Wikipedia mirror (wiki2)](https://wiki2.org/en/Guru_Logi_Champ)
- [Guru Logi Champ — The Cutting Room Floor](https://tcrf.net/Guru_Logi_Champ)
- [Guru Logi Champ — MobyGames](https://www.mobygames.com/game/20866/guru-logi-champ/trivia/)
