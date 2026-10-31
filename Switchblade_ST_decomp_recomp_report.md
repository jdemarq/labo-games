# Switchblade (Atari ST) — decompilation and recompilation report

Technical reference for the static recompilation of the **Atari ST version** of *Switchblade*
(Gremlin Graphics, 1989) to native code (C, then any target such as Android). It describes the
disk, the memory image, the code and data structures, the hardware the game uses, and how each
part was recompiled and validated.

Every address is an absolute 68000 address in the running game unless stated otherwise.

## 1. The game and its authors

| | |
|---|---|
| Title | **Switchblade** (Atari ST version, 1989) |
| Design, program, graphics and sound | **Simon Phipps** |
| Music | **Ben Daglish** |
| Developer / publisher | **Core Design** / **Gremlin Graphics** |
| In-game credits ("special thanks") | Rob Toone, Greg Holmes, Bob Churchill, Terry Lloyd, Andy Green, Jeremy Smith, Dave Pridmore, Kevin Norburn and Jayne |
| Copyright string | "COPYRIGHT © 1989 SIMON PHIPPS" |

The game is a flip-screen platform adventure of 128 screens. You play Hiro, the last of the
Bladeknights, who must rebuild the Fireblade in the UnderCity. The game data remains the
property of its authors.

## 2. Disk image and loading

**Source image:** an original Gremlin disk preserved in **Pasti** format (`.STX`).
- Single-sided, 82 tracks, 10 sectors per track. The BPB declares 800 sectors.
- Tracks 80 and 81 are empty. Track 79 holds the copy protection: one sector with a CRC error
  and a long track.
- The FAT12 filesystem contains a single file, **`SBLADEST.DAT`**:
  - size 316,590 bytes (`$4D4AE`);
  - SHA-1 `8694c71a937f301eab29501bc0b0bbd7e3dc2aed`;
  - a short script extracts it (see below).

**Reading the STX container** (little-endian):
- File header, 16 bytes: magic `RSY\0`, track record count at offset 10.
- Then one record per track: a 16-byte header `<u32 record size, u32 fuzzy-mask size, u16 sector
  count, u16 flags, u16 MFM size, u8 track number (bit 7 = side), u8 revolutions>`.
- Flags bit 0 clear: the sector data (512 bytes each, sectors 1…n) follows the header directly.
- Flags bit 0 set: n sector descriptors of 16 bytes follow `<u32 data offset, u16 bit position,
  u16 read time, u8 ID track, u8 ID side, u8 ID sector, u8 ID size, u16 CRC, u8 FDC status, u8
  reserved>`, then the fuzzy mask; sector data offsets are relative to the end of the fuzzy mask.
  Skip descriptors whose FDC status has bit 4 (record not found). Size = 128 << (ID size & 3).
- The next record starts `record size` bytes after the current one.

**Reading FAT12:** rebuild a flat image (track × 10 + sector − 1) × 512, read the BPB at offset
11 (`<u16 bytes/sector, u8 sectors/cluster, u16 reserved, u8 FAT count, u16 root entries, u16
total sectors, u8 media, u16 sectors/FAT>`), then follow the 12-bit cluster chain of the root
directory entry (first cluster at offset 26, size at offset 28; data starts after the root
directory, cluster 2 first).

**Boot sector:** the boot sector (offset `$1E`) is executable and does the following:
1. `Vsync`, then set colour 0 to black.
2. `Fopen("SBLADEST.DAT")`, `Fread($4D4AE bytes → $10000)`, `Fclose`.
3. Wait 250 VBLs.
4. `jmp $10000`, still in supervisor mode with the TOS stack.

**The loaded image:** `SBLADEST.DAT` is a raw memory image loaded at **`$10000`**. It starts with
a PRG-like header (`$601A …`), but the header is never interpreted: the first word is `bra.b $1001C`.

## 3. Copy protection (`$1001C`–`$1014C`)

The protection talks directly to the WD1772 floppy controller through the DMA ports `$FF8604/6`.
1. Select drive A by writing to port A of the YM sound chip.
2. Seek to track 79 (`$4F`).
3. Issue *Read Track* (`$E4`).
4. Measure the DMA byte counter (`$FF8609/B/D`) and compare the difference with `$3C`.

On failure the routine loops forever, flashing colour 0. On success it executes `moveq #7,d1;
bsr $10110` (deselect the drives) and then **`jmp $5D218`**, the real entry point.

The recompiled version simply starts at `$5D218`, with SR = `$2300` and SSP = `$7F00`. The
protection code is never executed and never needs to be recompiled.

## 4. Memory map

| Range | Contents |
|---|---|
| `$000000–$0003FF` | 68000 vectors. The game installs VBL `$70→$54EFA`, ACIA/IKBD `$118→$52BDA`, MFP Timer A `$134→$51218` |
| `$000400–$000FFF` | TOS system variables (only `_v_bas_ad`, `_hz_200` and `phystop` are emulated) |
| `$010000–$01014F` | Header and copy-protection loader |
| `$010150–$04F24B` | Data: sprites, music, samples, tables, and the world (see §6): |
| `$026958–$026E07` | World grid: 30 × 20 words, screen number or `$FFFF` |
| `$026E08–$034E07` | The 128 screens, `$1C0` bytes each (tile words + object slots) |
| `$034E08–$03EE07` | 256 background tiles of 16 × 16 pixels, 160 bytes each (247 are used) |
| `$03EE08` | Font |
| **`$04F24C–$05D4AE`** | **Code**: 9,373 instructions in 2,209 basic blocks, mixed with small data tables |
| `$065F6A` (approx.) | Supervisor stack in use during gameplay |
| `$0662A0` | Collision map of the current screen (one word per tile column and pixel row, 40 bytes per row) |
| `$0664A8`, `$066C78` | Work buffers (ending sequence, sprite restore) |
| `$068780` | Off-screen copy of the play area (320 × 176), copied to line 12 of the visible screen |
| `$070000`, `$078000` | The two screen buffers (double buffering, see §7) |
| `$0F8000` | TOS screen at boot (not used by the game) |

The game assumes a 1 MB machine (`phystop = $100000`) and runs only in ST low resolution (320×200,
16 colours).

## 5. Important code and data addresses

| Address | Description |
|---|---|
| `$5D218` | Real entry point: initialisation, then the main loop |
| `$54EFA` | **VBL handler**: increments the VBL counter `$54F5E`, sets `$54F5F = $FF`, updates music and palette |
| `$54F36` | "Wait for frame": waits until `$54F5E ≥ $54EE8` (target frame count), then clears it. The busy loop is at `$54F44/$54F4A` |
| `$54F62–$54F92` | Swaps the screen buffers: toggles bit 7 of `$54EEC` and writes `$54EEB/$54EEC` to the shifter registers `$FF8201/3` |
| `$54FB2` | Copies 16 palette words from `(a0)` to `$FF8240` |
| `$54F94` | XBIOS `Setscreen(-1,-1,0)`: the **only TOS call** in the game (`trap #14`) |
| `$52BA6` | IKBD init: sends `$12` (mouse off) and `$14` (joystick event mode), installs `$52BDA` |
| `$52BDA` | ACIA interrupt: a `$FE` or `$FF` header means the next byte is joystick 0 or 1. Clears ISRB bit 6 itself (MFP software end-of-interrupt) |
| `$52BA2` / **`$52BA3`** / **`$52BA4`** | Joystick 0 / **joystick 1** (bit 7 fire, bits 0–3 up/down/left/right) / **last key scancode** |
| `$5D392…$5D3D0` | Keys used: `$01` (Esc) and `$19` (P = pause) |
| `$55F46` | Keys `1`–`5` (scancodes 2–6) copy a parameter set, only when the flag `$5AA82` is set (debug/cheat) |
| `$4F24C` | **Music driver API**: a table of `bra.w` instructions (`+0`, `+4`, `+$A`→`$4FD5E`, `+$E`, `+$12`) |
| `$4F8AA/$4F8CC/$4F8EE` | The 3 music channel structures (`a6`). **`22(a6)` holds a code pointer** (`$4FBFC` or `$4FC70`) executed by `jmp (a0)` at `$4FBFA` |
| `$51000–$51A00` | Sound effects / digitised sample player. The second channel set has a code pointer at `16(a6)` (`$51864`), executed by `jmp (a0)` at `$51862` |
| `$51218` | **Timer A handler**, the digitised-sound player. It reads one byte per interrupt from the pointer at `$519BE`; value 0 ends the sample. It looks up a long in a table and writes `move.l` to `$FF8800` three times (YM registers 8/9/10 = volume DAC); `$400`/`$800` are the table offsets for the other two channels |
| `$5115A–$51178` | Starts a sample: `$511F2` → pointer, `$511F6` → TACR, `$511F7` → TADR, `$511F8/$511FA` → busy/loop flags |
| **`$5C3A2`** | **Object behaviour table**: 149 longs (indexes `$00`–`$94`), called by `jsr (a4)` at `$5B092` and `$5B14A`. The index byte `2(a5)` is checked with a signed `ble #$71`, so `$80`–`$94` are reachable too. Entry `$95` and beyond are data |
| `$5C56A` | Same table offset by `$72` entries (`jsr (a4)` at `$5B50A`): bonus and score items |
| **`$5C5F6`** | **Object descriptors** for types `$46`–`$4D` (routine `$5B852`): 8 records of `$56` bytes. Offsets `$00–$2C` are animation pointers, `$30–$36` are words. **Field `$42` is a code pointer** (`$5C8A6`) called by `jsr (a4)` at `$5BB28` |
| `$5D48A` | In-game palette (16 ST colour words), set at `$5D238` (start-up) and `$55B34` |
| `$5561C` / `$5561E` | Current screen column / row in the world grid; `$55620` = screen number, `$55622` = pointer to its record |
| `$556CC` | Draws the current screen into `$68780` and its collision map into `$662A0`, then shows it |
| `$58D46–$58E46` | Screen flips: moves `$5561C`/`$5561E` by ±1 when the player leaves the screen, then `$55A5C` |
| `$55692` | Reveals a zone: sets bit 15 of every tile of the current screen whose zone bits equal `$55626` |
| `$5562C` | New game: clears bits 14–15 of every tile of the 128 screens and selects the start screen |
| `$5B0A0`, `$5B178` | Spawn objects from the screen's object slots (on entering the screen / when a zone is revealed) |
| `$56030` | Ending text ("THE DAWN HAS COME …"), then waiting for fire |

**Complete list of indirect jumps.** These six are all of them. Keep the list in the recompiler
and make it fail if a new one appears:
- `$4FBFA` and `$51862` (music channel handlers);
- `$5B092`, `$5B14A` and `$5B50A` (object table);
- `$5BB28` (descriptor field `$42`).

**Other properties of the code:**
- There is no self-modifying code: write tracking in the reference emulator showed no writes to
  executed bytes.
- No code is copied or relocated.
- The only privileged operations are `move sr`, `ori/andi #…,sr` and `rte`.
- The game never reads the video counter, `_hz_200` or the MFP timer data registers, so its logic
  depends only on the VBL count and the joystick input.

## 6. World map, screens and tiles

The whole UnderCity is stored **uncompressed and static** in the image. The game changes only the
flag bits of the tile words while playing (and clears them for a new game), so the world can be
drawn straight from `SBLADEST.DAT` (30 × 20 screens of 320 × 176 pixels = a 9600 × 3520 image).

**World grid** at `$26958`: 20 rows of 30 words (`$3C` bytes per row). Each word is a screen number
(0–127) or `$FFFF` for no screen. `$559C6` reads `grid[$5561E][$5561C]`. All 128 screens are used
once. A new game starts on screen 0 at column 0, row 0: the surface, which is row 0 of the
grid; everything below is underground.

**Screen record** at `$26E08 + screen × $1C0` (448 bytes):

| Offset | Contents |
|---|---|
| `0–439` | 220 tile words, 20 columns × 11 rows, row by row |
| `440` | Unknown byte (not read by the tile or object code) |
| `441–447` | Object slots 1–7: bit 7 = spawn when the screen is entered; bits 0–6 = object type, an index into the behaviour table `$5C3A2` |

**Tile word:**

| Bits | Meaning |
|---|---|
| 0–8 | Tile graphic number (`$34E08 + n × 160`) |
| 9–10 | Reveal zone: entering a zone reveals all tiles of the screen with the same zone (`$55692`) |
| 11–13 | Object slot 1–7 (0 = no object); the object spawns at x = column × 16, y = row × 16 + 12 |
| 14 | When set, both spawners skip the tile (presumably: object already taken or destroyed); cleared for a new game |
| 15 | Revealed: unrevealed tiles are drawn black |

**Tile graphic** (160 bytes): 16 rows of 4 interleaved bitplane words (128 bytes, the ST low
resolution layout for 16 pixels), then 16 words of collision mask, one per row, copied to
`$662A0`.

**Drawing a screen** (`$556CC`): for each tile, revealed ones copy their 16 rows into the
off-screen buffer `$68780` (160 bytes per line) and their mask words into `$662A0`; unrevealed
ones write zeros. The 176 lines are then copied to line 12 of the screen buffer (the score bar
uses lines 0–11 and the status bar the lines below the play area).

## 7. Hardware used by the game

| Device | Use |
|---|---|
| Shifter | Palette `$FF8240–5F` (3-bit ST colours), screen base `$FF8201/03`, which double-buffers between `$70000` and `$78000`, `$FF820A` read once. No raster splits and no Timer B / HBL |
| YM2149 | Music is written in the VBL. Digitised effects use the volume registers driven by **MFP Timer A** in delay mode (`IERA`/`IMRA` bit 5, vector `$134`) |
| MFP 68901 | Timer A (samples). Timer C stays running at 200 Hz (TOS handler, HLE). ACIA on GPIP4 (channel 6, vector `$118`). Software end-of-interrupt (`VR` bit 3) |
| ACIA / IKBD | Joystick in event mode (`$FF` + state byte), keyboard scancodes |
| FDC / DMA | Only used by the protection (skipped) |
| TOS | `trap #14` `Setscreen` only. The vectors initially point to HLE stubs in a fake ROM at `$FC0000` |

## 8. Static recompiler

**Decoder.** A complete 68000 decoder, exact to the real chip: brief extension words ignore bits
8–10, for example. Cross-check it against an existing disassembler (Musashi's) over the whole
image before trusting it.

**Code discovery.** Recursive descent from the entry points, following branches, `bsr`/`jsr`,
`dbcc` and both sides of conditional branches, stopping at `rts`, `rte`, `jmp (An)` and illegal
opcodes. Labels go at branch targets, return addresses and after every SR change. Then add entry
points from these patterns and iterate until nothing new appears:
- `move.l #x,$vector` instructions (interrupt handlers);
- `pea` of code addresses;
- the pattern `lea label,An` / `move.l An,mem`;
- the jump table `$5C3A2` × 149 and the descriptor table `$5C5F6` (8 × `$56` bytes, field `$42`);
- *orphan routines*: plausible code right after an `rts`/`bra`, i.e. at least 3 valid, non-zero
  opcodes (at most 200) ending in an unconditional transfer, with less than 40% zero bytes;
- *data pointers*: any aligned long anywhere in the image whose value lands on plausible code
  (same test, at least 2 instructions, also accepted if it runs into already known code).

This finds all 9,373 instructions; execution traces from an interpreter are useful to confirm
it (every executed PC must already be known) but are not needed as input.

**Code generation.** Split the code into basic blocks of at most about 240 cycles and group them
into "region" functions (33 here). Shape of the generated C:
```c
uint32_t R_5B852(uint32_t pc) {          /* one region = several blocks */
  switch (pc) { case 0x5B852u: goto L_5B852; ... default: return bad_pc(pc); }
L_5B852: ;
  if (EVT()) return 0x5B852u;            /* event check at each block (interrupt latency) */
  CYC(26);                               /* static cycles of the block */
  { /* 05b852: cmpi.b #$0,(a5) */ uint32_t v = RB(C.a[5]); ... C.z = ...; }
  { /* 05b856: bne $5b8dc */ if (!C.z) { goto L_5B8DC; } CYC(2); }
  ...
}
```
- Branches inside a region become `goto`.
- Calls, returns, `jmp (An)` and cross-region jumps return the next PC to a **dispatcher**, which
  looks it up in a table indexed by `(pc - $10000) / 2` giving the region function.
- The 68000 stack stays in emulated RAM, so stack tricks behave exactly as on the real machine.
- Interrupts and exceptions build real 68000 stack frames, and `rte` pops them.
- Flags are exact, including the undefined N/V of `abcd`/`sbcd` (as Musashi computes them),
  divide overflow, and shift counts of 0 and ≥ 32.
- A "precise" mode with one block per instruction is useful for lockstep testing.

**Cycle model.** Musashi's 68000 cycle table, plus data-dependent costs: MULU 2 per set bit of
the source, MULS 2 per bit transition, shifts 2 × count, MOVEM 4/8 per register, Bcc/DBcc taken
or not, and fixed exception totals (34 for `trap`, 38 for divide by zero, 44 for an interrupt).

## 9. Runtime

- **CPU state:** registers, flags kept one per variable, big-endian memory access with I/O at
  `≥ $FF8000`, SR changes that swap USP/SSP, shift and BCD helpers.
- **TOS:** no ROM. The vectors point to stubs in a fake ROM area at `$FC0000`; the dispatcher
  runs C code for them: XBIOS `Setscreen` (plus `Setpalette`, `Supexec` for safety), BIOS
  `Setexc`, and the system VBL and Timer C handlers (`_hz_200`, `_vbclock`). Only `_v_bas_ad`,
  `_hz_200` and `phystop = $100000` are needed in the system variables.
- **Timing:** event-driven, time measured in CPU cycles: 8,010,613 Hz (PAL), 160,256 cycles per
  frame = 313 lines × 512 cycles, display from line 63 after the VBL.
  - MFP timers computed in MFP ticks (2.4576 MHz) so Timer A sample rates are exact.
  - The ACIA delivers one IKBD byte per scanline boundary, at least 21 lines apart.
  - The palette is latched per scanline; the frame is converted from 4 bitplanes to RGB at the end.
- **Sound:** YM2149 synthesis at 250 kHz (2 MHz / 8), 32-step envelope, 17-bit noise LFSR,
  cycle-stamped register writes, box-filtered down to 48 kHz mono.
- **Save states:** CPU registers + 1 MB RAM + hardware state (MFP, ACIA, YM registers).

## 10. Android host

- A game thread runs one frame per 20.005 ms (50 Hz), nudged by the audio queue level so sound
  never starves.
- Rendering in OpenGL ES 2, two passes: the 320 × 200 frame into an FBO at the chosen internal
  resolution (nearest, bilinear or sharp bilinear), then to the screen with optional FXAA and
  scanlines, in the chosen aspect ratio (4:3, 16:10 square pixels, stretch, integer, 16:9).
- Input: touch stick and buttons, gamepad and keyboard mapped to the joystick bits and scancodes.
- The build extracts `SBLADEST.DAT` from the STX, recompiles it to C, and compiles that C with
  the NDK.

## 11. Validation

1. **Per instruction:** each of the 9,373 instructions runs from 16 random machine states in
   Musashi and as generated C. Registers, SR, PC, all of RAM, I/O writes **and cycle counts**
   must be identical. Result: 0 differences. Injecting deliberate bugs into the generated code
   confirms the test catches them.
2. **Lockstep:** the *same* hardware model driven by Musashi, against the precise recompiled
   build. The 1 MB of RAM is identical at every checkpoint over 4 × 90,000 frames of random play
   and a long walk through the level.
3. **Exploration:** several hours of headless random and directed play. The runtime halts with
   the PC if it ever reaches code that was not recompiled; random play executes about 78% of
   the instructions.
4. **Device:** install on an Android emulator, play with touch input, capture every rendering
   mode.

**Finding data structures:** diff RAM dumps taken just before and after an event. A screen
change, for instance, flips bit 15 in a run of 140 words at a `$1C0` stride, which leads to
the screen records, then to the code that reads them (§6).

## 12. Pitfalls met (read this before redoing it)

- **Use the original STX.** The cracked `.st` contains encrypted files and a broken PRG relocation table.
- **The real entry is `$5D218`**, not `$10000`, because the protection would fail.
- **Indirect calls through data structures:**
  - The object table has 149 entries, not `$72`: a signed compare lets indexes `$80`–`$94` through.
  - Descriptor field `$42` points to `$5C8A6`, a routine placed right after a data table. It was
    invisible to "after-rts" heuristics and caused a freeze in the level 1 basement.
  - Scanning all data longs that point to code fixed this class of bug.
- **IKBD timing:** the game's ACIA handler reads *one byte per interrupt*, so the ACIA must
  deliver the bytes separately (`$FF` and then the state byte), with a new edge on GPIP4 each time.
- **Musashi `m68k_set_reg(SR)` does not swap stacks**: manage the USP and SSP yourself.
- **Interrupt latency** matters for the Timer A samples (8–10 kHz). Put an event check at the start
  of every block and cap block length (about 240 cycles), giving a worst case of about 45 µs.
- **Cycle-exact comparisons** need Musashi's data-dependent MUL timings and fixed exception costs
  (34 for `trap`, 38 for divide by zero, 44 for an interrupt).
- **The Android NDK's clang** rejects implicit declarations: include `<string.h>` wherever
  `memset` is used.

## 13. Redoing the port: checklist

1. Extract `SBLADEST.DAT` from the original STX and check its SHA-1 (§2).
2. Load it at `$10000` in a 1 MB address space; start at `$5D218`, supervisor mode, SR `$2300`,
   SSP `$7F00`; provide HLE stubs for the vectors and for XBIOS `Setscreen` (§3, §7).
3. Discover the code from `$5D218` and the vector handlers; add the object table `$5C3A2` × 149,
   the descriptor table `$5C5F6` (field `$42`) and a scan of data longs pointing to code; check
   that only the six known indirect jumps remain (§5, §8).
4. Translate to C (or any target) with exact flags and the 68000 stack in emulated RAM; return
   to a dispatcher for every non-local transfer (§8).
5. Implement VBL (50 Hz, 160,256 cycles), MFP Timer A, the ACIA/IKBD joystick protocol, the
   shifter palette and screen base, and the YM2149 including volume-register sample playback
   (§7, §9).
6. Feed input through the joystick bytes the ACIA handler writes (`$52BA3`, fire = bit 7) and
   key scancodes (`$52BA4`); present `$70000`/`$78000` as selected by `$FF8201/03` (§5).
7. Validate instruction by instruction and in lockstep against an interpreter, then play long
   headless sessions (§11, §12).
