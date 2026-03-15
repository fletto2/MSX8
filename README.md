# MSX8 - Improved I/O Patcher

Forked from [lesbird/MSX8](https://github.com/lesbird/MSX8) - MSX ROM launcher for Z80 retro platforms (Heathkit H8, NABU PC, RC2014).

This fork improves the NABU BIOS patcher (MSX8P) with heuristics to reduce false positives when patching direct hardware I/O in MSX game ROMs.

## What's Changed (NABU BIOS patcher)

The original patcher scans ROM code byte-by-byte for MSX I/O port values and replaces them with NABU equivalents. The problem: it cannot distinguish code from data, so it may corrupt sprite tables, music data, or lookup tables that happen to contain port-value bytes.

This fork adds two improvements:

### 1. Z80 instruction boundary tracking

A 256-byte opcode length table (`Z80LEN`) and skip routine (`PATSKIP`) walk the ROM respecting actual Z80 instruction lengths. The scanner only checks bytes that fall on instruction boundaries, so operand bytes inside multi-byte instructions are never misidentified as I/O opcodes.

**Before:** `LD HL,$98D3` (`21 D3 98`) — scanner sees `D3 98` at offset+1 and falsely patches it as `OUT ($98),A`

**After:** scanner reads `$21` at offset 0, looks up length=3, skips to offset+3 — operand bytes correctly ignored

Handles all Z80 prefixes (CB, DD/FD, ED) including 4-byte DD CB and ED 43-7B variants.

### 2. Context-validated LD C patching

`LD C,$98` (`0E 98`) is common as data. The patcher now only patches `LD C,<port>` when a C-register I/O instruction is confirmed within 12 bytes ahead:

- **Individual I/O**: `OUT (C),r` / `IN r,(C)` (ED 40-79, pattern `AND $C6 == $40`)
- **Single-transfer block I/O**: OUTI/INI/OUTD/IND (ED A2-AB)
- **Repeat block I/O**: OTIR/INIR/OTDR/INDR (ED B2-BB) — **PSG only**

Repeat block I/O is rejected for VDP ports because OTIR causes visual snow on the TMS9918 — the repeat transfer is too fast for the VDP's internal timing. Real games use OUTI in a DJNZ loop instead, so `LD C,$98` + OTIR is almost certainly a data false positive.

### Additional improvements

- **Full PPI port handling**: OUT/IN to $A8 (slot select), $A9 (keyboard matrix), $AA (PPI-C row select), $AB (PPI mode) — the original patcher only handled $A8 and $A9
- **Smart PSG read**: RST 18 handler checks the selected PSG register — R14/R15 (joystick) routes through the NABU joystick translator, other registers do a real PSG read from NABU port $40
- **Keyboard matrix wrapper**: RST 10 handler masks the row number with $0F before passing to the keyboard emulator, fixing compatibility with games that use the read-modify-write PPI-C pattern
- **Fixed keyboard RST opcode**: was $DF (RST 18 = joystick handler), now $D7 (RST 10 = keyboard handler)

### Patch table

| MSX Port | Instruction | NABU Replacement | Method |
|----------|------------|-----------------|--------|
| $98 | OUT/IN (VDP data) | $A0 | byte patch |
| $99 | OUT/IN (VDP ctrl) | $A1 | byte patch |
| $A0 | OUT (PSG addr) | RST 28 + NOP | trampoline |
| $A1 | OUT (PSG data) | RST 30 + NOP | trampoline |
| $A2 | IN (PSG read) | RST 18 + NOP | trampoline |
| $A8 | OUT (slot select) | NOP NOP | no-op |
| $A8 | IN (slot read) | LD A,$00 | return 0 |
| $A9 | OUT (PPI-B) | NOP NOP | no-op |
| $A9 | IN (keyboard) | LD C,A + RST 10 | trampoline |
| $AA | OUT (PPI-C row) | NOP NOP | no-op (A preserved) |
| $AA | IN (PPI-C read) | LD A,$F0 | return upper nibble |
| $AB | OUT (PPI mode) | NOP NOP | no-op |
| LD C,$98-$A2 | + OUTI/OUT(C)/IN(C) | NABU port | context-validated |

### Known limitations

- **INC C/DEC C port toggle** (4 games): NABU PSG ports $40/$41 are reversed relative to MSX $A0/$A1, so incrementing C gives the wrong port
- **Inline data blocks**: if code jumps over data, the linear scanner may lose instruction boundary sync within the data region
- **Computed port values**: `OUT (C),r` where C is loaded from memory or computed at runtime cannot be statically detected (games using BIOS port indirection at $0006/$0007 work automatically since the NABU BIOS stores correct ports there)

## Original README

See the [upstream repository](https://github.com/lesbird/MSX8) for full documentation on MSX8 usage, building, and platform-specific setup (Heathkit, NABU, RC2014).
