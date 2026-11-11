# Guru Logi Champ (GBA) — data analysis for an Android port

Technical notes on an attempt to port the Game Boy Advance puzzle game *Guru Logi Champ* to a
native Android application: how the extracted data was analysed, what the analysis showed, how
the data is formatted, and what a working port requires.

## 1. The game and its rights holders

| Title | **Guru Logi Champ** (ぐるロジチャンプ), sometimes written *Guru Logic Champ* |
|---|---|
| Platform | Game Boy Advance, Japan only |
| Year | 2001 |
| Developer / publisher | **Compile** |
| Genre | Puzzle: build a target picture on a block board by firing blocks in and pulling them out, the board rotating so you can act from any side |

The game, its code, graphics, music, text and level data remain the property of their rights
holders. The Game Boy Advance header and boot logo belong to Nintendo.

This document contains **no** game data: no ROM bytes, graphics, text, level layouts or audio.
It describes only methods and data structures. Anyone reproducing the work must use their own
legally obtained copy of the cartridge, and should never commit ROM data to a repository.

## 2. Analysis method

The extraction pipeline produced two kinds of output: binary files named after asset types
(tiles, sprites, palettes, maps, music…) and JSON files meant to describe level metadata, tiles
and entities. Each was checked against the ROM rather than trusted by its name.

1. **Identity check.** Hash every binary (MD5) and group identical files.
2. **Header check.** Look at the first bytes of each file for the GBA cartridge header (entry
   branch at `0x00`, Nintendo logo at `0x04`, title at `0xA0`, fixed value `0x96` at `0xB2`).
3. **Prefix check.** Compare each smaller file with the start of the larger ones.
4. **Record reconstruction.** For each JSON record, concatenate its fields back into bytes and
   compare them with the ROM at the record's `index`.
5. **Content check.** Disassemble a sample of reconstructed records as ARM and Thumb code, and
   look at offsets, strides and value ranges for signs of real structures (bounded coordinates,
   small type enums, pointers into `0x08xxxxxx`).

## 3. Results

### 3.1 The binaries are ROM prefixes, not assets

- All binaries begin with the full GBA cartridge header.
- They fall into three groups of byte-identical files: 64 KiB, 1 MiB and 2 MiB.
- Each smaller file equals the start of the larger ones: every file is simply **the first N bytes
  of the same ROM**, whatever its name.

No asset was isolated: the names (`tiles`, `music`, …) do not match their content.

### 3.2 The JSON records are fixed-size slices of the ROM

Every record has the shape:

```json
{ "index": 264, "type": "XX", "coordX": "0xXXXX", "coordY": "0xXXXX", "flags": "0xXXXX", "data": "XXXXXXXXXX" }
```

| Field | Bytes | Meaning found |
|---|---|---|
| `index` | — | byte offset in the ROM, always a multiple of 12 |
| `type` | 1 | byte at `index` |
| `coordX` | 2 | next 2 bytes, written as a big-endian hex string |
| `coordY` | 2 | next 2 bytes, same |
| `flags` | 2 | next 2 bytes, same |
| `data` | 5 | last 5 bytes, as a hex string |

Concatenating the fields gives back exactly the 12 ROM bytes at `index`. The records therefore
cut the ROM into **12-byte windows** with no relation to the game's real structures:

- "metadata" records start at offset 0, i.e. inside the cartridge header;
- "entity" records disassemble cleanly as **ARM machine code** (data-processing and stack
  instructions), not as game objects;
- "tile" records are another filtered selection of the same windows.

The labels `type`, `coordX`, `coordY` and `flags` carry no meaning. Note also that GBA data is
**little-endian**, so even a real 16-bit field would be misread by this big-endian formatting.

### 3.3 Conclusion

No game format has been decoded yet. A renderer fed with this data can only draw noise; the
extraction step has to be redone from the real formats below.

## 4. Real GBA data formats to look for

| Data | Format |
|---|---|
| Colours | 15-bit **BGR555**, little-endian `u16` (`0bbbbbgg gggrrrrr`); palettes of 16 × 16 or 1 × 256 colours, 512 bytes for BG + 512 for sprites |
| Tiles | 8 × 8 pixels; **4bpp** = 32 bytes (two pixels per byte, low nibble = left pixel) or **8bpp** = 64 bytes |
| Text tilemaps | `u16` per cell: bits 0–9 tile number, 10 horizontal flip, 11 vertical flip, 12–15 palette bank |
| Affine tilemaps | `u8` per cell (tile number only); used for rotating/scaled backgrounds, a likely fit for the rotating board |
| Sprites (OAM) | 3 × `u16` attributes: Y, shape, mode / X, size, flips or affine index / tile, priority, palette |
| Pointers | 32-bit little-endian, ROM data at `0x08000000`–`0x09FFFFFF` |
| Compression | BIOS formats, identified by the first byte of a 4-byte header (`type`, then 24-bit uncompressed size): `0x10` LZ77, `0x2x` Huffman, `0x30` RLE, `0x8x` diff filter; called through `SWI 0x11/0x12` (LZ77), `0x13` (Huffman), `0x14/0x15` (RLE) |
| Text | Japanese game: Shift-JIS or a custom table mapping to font tiles |
| Puzzles | Unknown, game-specific; expect a small grid (target picture + starting blocks) per level, reached through a pointer table |

Memory map useful to follow data at run time: EWRAM `0x02000000`, IWRAM `0x03000000`, I/O
`0x04000000`, palette RAM `0x05000000`, VRAM `0x06000000`, OAM `0x07000000`.

## 5. How to locate the real data

1. **Disassemble** the ROM in Ghidra (ARM7TDMI, mixed ARM/Thumb; switch mode on `BX` targets).
   Find the entry point, the interrupt handler and the main loop.
2. **Find decompression calls** (`SWI 0x11`–`0x15`) and the pointers passed to them: each leads
   to a compressed asset.
3. **Watch the hardware** in a debugging emulator (mGBA, No$GBA): set write breakpoints on
   palette RAM, VRAM and OAM while a puzzle loads, then trace back to the ROM source address.
4. **Scan for pointer tables**: runs of little-endian `u32` values all in `0x08xxxxxx`, evenly
   spaced, usually point to levels, graphics or text.
5. **Validate visually**: render a candidate palette + tile block + tilemap and compare it with
   an emulator screenshot of the same scene.

## 6. Two ways to build the Android port

**A. Static recompilation** (as for *Switchblade*): translate the ARM/Thumb code to C, build it
with the NDK, and supply a runtime for the PPU (tile modes, affine backgrounds, sprites), DMA,
timers, interrupts, keypad, Direct Sound + PSG audio and save memory. Exact behaviour, faster to
reach.

**B. Re-implementation**: decode every format of section 4, rewrite the rules (firing and
pulling blocks, board rotation, win check) natively, and load graphics and levels from the
user's own ROM at first launch. Cleaner code, but every format must be decoded.

In both cases, the app should read data from a ROM supplied by the user (checked by size, header
and checksum) and never ship it.

## 7. Checklist

1. Write a ROM loader that verifies the user's file without distributing it.
2. Map code and data with the steps of section 5.
3. Write one extractor per real format, each validated against emulator screenshots.
4. Choose approach A or B and validate frame by frame against an emulator.
5. Android host: landscape, integer scaling of the 240 × 160 frame, touch controls mapped to the
   GBA buttons.

## Sources

- [Guru Logi Champ — Wikipedia mirror (wiki2)](https://wiki2.org/en/Guru_Logi_Champ)
- [Guru Logi Champ — The Cutting Room Floor](https://tcrf.net/Guru_Logi_Champ)
- [Guru Logi Champ — MobyGames](https://www.mobygames.com/game/20866/guru-logi-champ/trivia/)
- [GBATEK — GBA technical reference](https://problemkaputt.de/gbatek.htm)
