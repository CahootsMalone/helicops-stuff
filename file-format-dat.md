# Format of Helicops DAT Files

[Helicops](https://en.wikipedia.org/wiki/Helicops_(video_game)) has 23 files ending with the extension `DAT`. Each DAT file contains a set of related files (listed in a table at the start of the DAT file). DAT files with names ending in "16" correspond to levels in the game; the remaining DAT files appear to contain general assets, although `PIC.DAT` contains a file for each level. 

## Header

Each DAT file starts with a header. The header consists of 20 bytes of metadata followed by a table of files contained in the DAT file.

Offset | Type | Length | Description
---|---|---|---
0 | char | 8 | ISVPATAD (DATAPVSI; PVSI stands for Paragon Visual Systems Inc., the game's developer)
8 | uint | 4 | Unknown (always 2; could be file format version)
12 | uint | 4 | Unknown (0 or 1; perhaps whether or not the data are compressed as it's only 0 in `SOUNDS.DAT`)
16 | uint | 4 | File count (N<sub>file</sub>)
20 | File table entries (see below) | 44 * N<sub>file</sub> | Start of table of files; contains N<sub>file</sub> entries

### File Table Entry

Each file table entry is 44 bytes long. Entries are "encrypted" with an [XOR cipher](https://en.wikipedia.org/wiki/XOR_cipher); the key is `0xAA` (the number of null/zero bytes in each entry makes this obvious).

Each entry contains the following information (the offsets in the table below are relative to the start of each entry):

Offset | Type | Length | Description
---|---|---|---
0 | char | 32 | Null-terminated file name; trailing bytes are set to zero
32 | uint | 4 | Offset (absolute, i.e., relative to the start of the DAT file)
36 | uint | 4 | Uncompressed length
40 | uint | 4 | Compressed length (0 if data aren't compressed)

For example, the first entry in `PIC.DAT` is:

```
E7 C3 D9 D9 9B 9B 84 FA E3 E9 AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA A2 AE AA AA D0 A4 AB AA 2F 94 AA AA
```

XORing each byte with `0xAA` yields:

```
4D 69 73 73 31 31 2E 50 49 43 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 08 04 00 00 7A 0E 01 00 85 3E 00 00
```

This gives:
* File name: Miss11.PIC
* Offset: 1032
* Uncompressed length: 69242
* Compressed length: 16005

As expected, the sum of the starting offset (1032) and compressed length (16005) yields 17037, the starting offset of the next file (reported in the next entry in the table).

See [this file](/scripts/helicops-file-summary.md) for the contents of the file tables across all DAT files.

## Compression

*This section is just rough notes for now.*

* TBD: some form of compression (possibly a form of [run-length encoding](https://en.wikipedia.org/wiki/Run-length_encoding)) is used for most data. Certain bytes clearly contain compression-related information, but how they're meant to be interpreted isn't obvious; they might be obfuscated or correspond to hard-coded instructions in the game's executable.
* `0xFF` (when it appears as a non-data value) is consistently followed by eight uncompressed bytes; this is obvious in some of the grayscale ramps within the colour palettes (files with the extension "ACT" - Adobe Color Table - and an uncompressed length of 768 bytes, i.e., 256 colours with one byte for each channel) in `GRAPHICS.DAT` and in some of the plaintext sections of the files in `PIC.DAT`.
* `0xF7` is followed by three bytes.
* `0xFF` XORed with `0xAA` gives `0x55`, which isn't obviously significant.
* To get `0x08` from `0xFF`, it can be XORed with `0xF7`, but XORing other apparently-non-data values with `0xF7` doesn't yield anything that seems meaningful.
* It seems likely that compression information is stored in bits within a given byte (perhaps some bits specify the number of bytes to follow, other bits specify the type of compression instruction, etc.).
* Run-length encoding seems more likely than index-based compression (the use of a lookup table for repeated sequences); it's fairly clear that some of the less-compressed colour palettes (Adobe Colour Table files identified by the extension "ACT") in `GRAPHICS.DAT` don't contain any sort of LUT (see sample data below).
* Parts of the plaintext data in `PIC.DAT` appear to be split up, e.g., the level names (see sample data below), so there are presumably instructions for jumping forward and backward within a file.

### Sample Data

I've split these up a bit, primarily based on (what appear to be) non-data `0xFF` bytes.

#### Samples from GRAPHICS.DAT

Name | Start Offset | Length (Uncompressed) | Length (Compressed)
--- | --- | --- | ---
help1.act | 13830651 | 768 | 832

Starts with lots of sets of 3 values that are the same (grayscale ramp).

```
FE ED F0 // ?
10 10 10 18 18 18 21 
FF 21 21 31 31 31 42 42 42 
FF 52 52 52 63 63 63 73 73 
FF 73 84 84 84 8C 8C 8C 9C 
FF 9C 9C AD AD AD BD BD BD 
FF CE CE CE DE DE DE EF EF 
FB EF FF 1F 00 DE DE FF CE CE // Start of X X FF sets (DE DE FF, CE CE FF, C6 C6 FF, etc.)
FF FF C6 C6 FF BD BD FF AD 
FF AD FF 9C 9C FF 8C 8C FF 
FF 84 84 FF 73 73 FF 6B 6B 
FF FF 63 63 FF 52 52 FF 42 
FF 42 FF 31 31 FF 21 21 FF 
FF 10 10 F7 00 00 FF 00 00 
FF DE 63 00 EF B5 5A E7 94 
FF 00 EF 9C 00 AD 8C 00 F7 
FF DE 00 FF EF 00 42 42 00 
FF D6 D6 00 E7 E7 00 EF EF 
FF 00 F7 F7 00 FF FF 00 B5 
FF FF FF AD FF FF A5 FF FF 
EF 94 EF EF 9C 88 00 FF FF 7B 
FE 7E 00 CE D6 00 EF F7 00 F7 
FF FF 00 EF FF 00 8C 9C 00 
FF E7 FF 00 DE FF 00 94 AD 
FF 00 D6 FF 5A DE FF 21 63 
FF 73 00 C6 FF 00 6B 8C 00 
FF 6B 94 00 B5 F7 00 BD FF 
FF 00 73 A5 00 7B AD 00 84 
FF BD 00 8C C6 00 94 CE 00 
FF 9C DE 00 A5 E7 00 AD F7 
FE 80 00 42 AD DE 00 21 31 00 
FF 4A 6B 00 6B 9C 00 7B B5 
FB 00 8C D4 00 E7 00 A5 EF 00 
FF AD FF 52 C6 FF 21 52 6B 
FF 00 52 7B 00 63 94 00 6B 
FF A5 00 73 AD 00 7B BD 00 
FF 84 C6 00 8C D6 00 94 DE 
FF 39 84 AD 42 9C CE 39 94 
FF C6 18 42 5A 00 5A 8C 00 
FF 63 9C 00 73 B5 00 94 E7 
FF 00 9C F7 00 A5 FF 29 42 
FF 52 31 63 84 52 BD FF 4A 
FF B5 F7 10 29 39 00 29 42 
E3 00 42 E9 00 0D 10 D1 00 EF 00 9C 
FF FF 21 39 4A 42 73 94 18 
F7 31 42 18 2D 10 18 29 00 31 
FF 52 00 52 8C 00 94 FF 18
FF 39 52 00 39 63 00 4A 84 
FF 00 7B D6 00 8C F7 00 39 
BF 6B 00 42 7B 00 52 07 10 C6 
FF 00 7B E7 00 84 EF 00 8C 
FF FF 18 29 39 10 21 31 08 
FF 21 39 00 73 DE 00 84 FF 
FF C6 CE D6 29 31 39 5A 7B 
FF 9C 29 39 4A 31 52 73 29 
FF 4A 6B 21 42 63 18 39 5A 
FF 10 29 42 10 31 52 08 29 
FF 4A 00 08 10 00 10 21 00 
FF 18 31 00 21 42 00 29 52 
FF 00 31 63 00 39 73 00 73 
FF E7 00 7B F7 00 63 CE 00 
FF 63 D6 00 6B DE 10 31 5A 
FF 00 21 4A 00 29 5A 00 42 
FF 94 00 4A A5 00 5A C6 8C 
FF A5 C6 73 8C AD 52 6B 8C 
FF 31 42 5A 42 5A 7B 31 4A 
FF 6B 29 42 63 21 39 5A 18 
FF 31 52 10 21 39 10 29 4A 
FD 08 CF 10 18 39 00 39 84 00 
FF 42 9C 00 4A AD 00 52 BD 
FF 00 52 C6 00 5A CE 00 5A 
FF D6 00 63 E7 00 6B F7 18 
FA 48 10 21 D3 10 7B 00 39 8C 00 
7F 42 A5 00 4A B5 00 4A 2D 20 
FE 33 20 DE 39 52 7B 21 31 4A 
DF 29 42 6B 18 31 BB 10 52 08 
FE CC 10 10 29 00 29 6B 00 31 
FF 84 00 39 94 00 42 AD 18 
FA E5 00 18 3F 20 5A 00 29 73 18 
BF 29 4A 10 21 42 08 20 20 10 
FF 31 10 18 29 08 10 21 18 
7D 29 15 20 4A 18 21 39 21 BD 10 
FF 18 31 08 10 29 18 21 42 
FF 42 4A 6B 18 21 4A 10 18 
FF 39 31 31 52 21 21 39 29 
BF 29 4A 18 18 31 21 9B 20 18 
F6 BA 20 63 10 6B 20 00 08 00 00 
FF 21 10 08 31 18 08 52 21 
FF 08 6B 10 00 39 21 00 29 
FF C6 00 4A 5A 00 10 BD 00 0F 18 FF FF FF
```

Name | Start Offset | Length (Uncompressed) | Length (Compressed)
--- | --- | --- | ---
keymap.act | 14392490 | 768 | 800

```
// F8 at the end of this sequence is clearly first byte of a triple continued after 0xFF
8A ED F0 80 EE F2 80 F1 F2 F5 F0 F6 F0 C0 BE 03 00 DC C0 A6 CA F0 ED F0 F8 
FF F8 F8 F7 F7 F7 F6 F6 F6 
FF D2 FC FF EF EF EF EE EE 
FF EE E9 E9 E9 B2 FF FF F5 
FF FF 26 E4 E4 E4 E3 E3 E3 
FF E2 E2 E2 F4 FF 12 E1 E1 
FF E1 E0 E0 E0 90 FF FF F2 
FF CD CD D7 D7 D7 D3 D3 D3 
FF D2 D2 D2 D1 D1 D1 D0 D0 
FF D0 CD CD CD C3 C3 C3 BE 
FF BE BE BD BD BD 55 E2 FF 
FF BA BA BA 00 FF FF B0 B0 
FF B0 00 F9 FF 00 FA F9 AE 
FF AE AE AD AD AD AC AC AC 
FF 4D CC F9 AA AA AA A9 A9 
FF A9 00 EC FF A6 A6 A6 4C 
FF C9 DE E2 82 82 9D 9D 9D 
FF 9B 9B 9B 00 D6 FF 99 99 
FF 99 98 98 98 00 FF 00 00 
FF FC 00 00 CB FF 93 93 93 
FF 00 F6 00 00 F3 00 00 C0 
FF FF 8D 8D 8D 00 C0 FB 8C 
FF 8C 8C 11 BE CD 8B 8B 8B 
FF 8A 8A 8A 01 E8 01 89 89 
FF 89 00 E8 00 88 88 88 01 
FF B7 F7 87 87 87 85 85 85 
FF 82 82 82 00 AD ED 00 A8 
FF FD 7C 7C 7C 34 94 BD 00 
FF A3 F5 77 77 77 00 9C EA 
FF 06 9B D6 72 72 72 00 90 
FF DC 00 8B F6 6D 6C 6C 00 
FF 90 CB 6B 6B 6B 69 69 69 
FF C7 3F 3F 67 67 67 66 66 
FF 66 00 87 CB 00 87 C0 64 
FF 63 63 00 7D DD 00 78 EF 
FF 00 7D CB 04 A0 04 60 60 
FF 60 03 9F 03 00 7D C0 5B 
FF 5B 5B 03 98 03 00 76 B7 
FF 03 96 03 58 58 58 00 71 
FF C0 05 77 92 01 71 A8 54 
FF 54 54 00 63 E7 07 72 7F 
FF 00 63 DF 4F 4F 4F 4E 4E 
FF 4E 00 63 AD 00 59 DF 4D 
FB 4D 4D 9D 00 00 59 D6 4C 4C 
FF 4C 4B 4B 4B 4A 4A 4A 49 
FF 49 49 00 5F 96 00 55 CB 
FF 48 48 48 47 47 47 46 46 
FF 46 E7 00 00 45 45 45 44 
FF 44 44 00 4F BB 43 43 43 
FF DB 00 00 D8 00 00 41 40 
FF 40 D5 00 00 3F 3F 3F D0 
FE ED F0 4D 9A 00 49 AD 00 51 DF 82 3E 3E 3E CD A1 00 00 00 
FF 02 51 70 3C 3C 3C C6 00 
FF 00 3A 3A 3A 38 38 38 00 
FF 41 9B 37 37 37 B6 00 00 
7F 36 36 36 35 35 35 AA ED F0 
FF 3C 8B A8 00 00 32 32 32 
EF A5 00 00 A2 ED F0 3B 77 9F 
FF 00 00 03 3B 68 2F 2F 2F 
FF 9B 00 00 9A 00 00 98 00 
FF 00 2D 2D 2D 95 00 00 93 
FF 00 00 91 00 00 90 00 00 
FF 2B 2B 2B 8E 00 00 8D 00 
FF 00 8C 00 00 89 00 00 29 
FF 29 29 87 00 00 28 28 28 
FF 84 00 00 83 00 00 27 27 
EF 27 30 17 63 F1 F0 26 26 26 
FF 7D 00 00 78 00 00 77 00 
FF 00 08 28 51 74 00 00 71 
BF 00 00 22 21 21 6E ED F0 28 
FF 51 6C 00 00 20 20 20 0A 
FF 1E 65 69 00 00 1F 1F 1F 
FF 67 00 00 1E 1E 1E 04 20 
DF 4A 1C 1C 1C 5C ED F0 20 41 
FF 1A 1A 1A 19 19 19 18 18 
FF 18 17 17 17 07 18 38 16 
FF 16 16 08 17 30 15 15 15 
DF 10 10 38 15 14 8C 21 17 0E 
BF 29 15 08 4D 14 13 98 22 12 
AA 9E 22 11 A4 22 10 AA 22 0F B0 22 0D EA B6 22 0C BC 22 0B C2 21 09 0C 09 FB 0B 0A CB 21 FF FB F0 A0 A0 2F A4 80 80 80 62 11 FF 63 00 DA 21 00 DD 20 63 00 E9 20
```

Name | Start Offset | Length (Uncompressed) | Length (Compressed)
--- | --- | --- | ---
NAG.act | 14974307 | 768 | 820

```
FF 02 0D 07 00 29 08 04 10 
FF 1F 00 31 08 00 35 10 00 
FF 31 18 00 33 24 00 00 42 
FF 00 08 39 00 08 42 08 04 
FB 3D 08 0A 00 10 39 08 10 42 
FF 00 1F 3E 1E 0F 0A 1A 26 
FF 11 1A 1E 2E 2F 25 2B 0C 
FF 34 1D 22 32 20 1E 31 37 
FF 31 31 31 41 1B 0E 41 30 
FF 15 37 2D 35 47 2C 2D 3F 
FF 39 0A 39 31 39 46 31 39 
EF 00 00 4A 08 49 00 08 46 04 
FF 26 45 08 08 4A 08 20 4A 
FF 06 1E 50 06 22 75 10 10 
FF 42 10 1C 42 26 27 42 10 
FF 18 4A 10 20 64 13 27 62 
FF 25 2C 63 0E 16 90 09 22 
FF 90 08 29 94 10 29 90 08 
FF 31 8C 10 31 8C 0A 2B 99 
FF 18 25 92 10 31 94 10 31 
FF 9C 18 31 8C 18 31 94 1E 
FF 2B 94 25 31 90 0C 42 12 
FF 00 39 2D 18 39 30 35 39 
DF 29 39 39 31 31 AB 00 39 42 
FF 45 3F 0E 46 39 25 46 39 
F3 2D 4A 42 00 C0 01 42 42 39 3D 
FF 4A 39 39 00 47 1E 01 45 
FF 32 1E 46 2B 25 45 42 21 
FF 5F 23 02 62 42 27 64 3E 
FF 45 42 23 4A 42 31 42 42 
FF 3D 4A 42 39 42 50 23 42 
FF 52 35 41 6B 2A 3D 82 3A 
FF 12 45 62 2F 47 63 0D 45 
FF 86 25 45 84 46 39 46 3F 
FF 4B 46 3D 47 6A 10 39 94 
FF 10 39 9C 21 39 8C 19 3F 
FF 96 21 44 8E 21 41 9C 31 
FF 4A 95 00 5A 4A 04 5C 4E 
FF 08 63 4A 00 72 4D 12 5C 
FF 4C 2D 5E 4A 09 6D 4E 29 
FF 79 4B 07 63 60 2C 63 60 
FF 1D 60 8C 3A 63 8D 22 8C 
FF 68 35 7A 98 18 C2 A8 59 
FF 2C 15 56 4A 1F 56 4A 33 
FF 4F 41 3F 4C 42 4A 5C 3D 
FF 40 5C 4A 3E 56 5D 18 54 
FF 67 31 52 50 41 56 68 3F 
EF 4A 4A 52 5A 71 10 65 4C 76 
FF 2D 1B 77 4E 29 72 5E 1E 
FF 76 56 49 71 74 15 6F 68 
FF 3E 71 79 3C 92 3C 20 8D 
FF 66 24 C2 38 1E BD 65 22 
FF 8A 7B 11 89 7A 39 A0 7B 
FF 27 C9 7C 26 4A 4E 5A 52 
FF 4A 5A 5D 46 53 57 53 54 
FF 54 5A 58 63 5A 52 56 63 
FF 54 52 72 56 59 4A 60 63 
FF 5A 63 57 63 5D 53 73 61 
FF 51 56 7C 52 76 83 52 73 
FF AB 6F 4A 54 6F 5E 56 69 
FF 73 56 6D 49 63 67 71 63 
FF 69 68 68 67 76 8A 7E 5B 
FF 5A 79 7B 60 7A 74 76 77 
FF 7B 93 8A 71 60 88 81 88 
FF 97 6F 63 C3 74 67 77 98 
FF 31 97 95 28 AB 92 1D A8 
FF A7 27 74 8C 72 6D 9E 7C 
FF A6 9C 69 67 97 A8 87 A0 
FF A9 97 95 9F 96 A7 A6 A8 
FF 9C 9A A5 A8 A4 A4 AD A9 
FF A4 B9 B0 C1 A3 19 DE A6 
FF 15 BC A1 44 DE A4 40 CA 
FF C5 29 EB D8 24 D6 CB 48 
FF F4 E1 45 D7 A5 72 B5 AC 
FF A7 DE AA A3 E6 D8 5A FF 
FF ED 5A E3 D4 76 E9 D5 99 
FF 65 8C BE 56 95 C7 71 A9 
FF CA 93 B2 C7 A9 B8 C9 A6 
FF CB D7 AD D6 F0 BD B9 B9 
FF C7 BB BB BF CA BF CC CA 
FF BB C4 C9 CE BE D1 E4 BD 
FF E2 F3 D4 CA BF D1 D3 D9 
FF EF C9 C2 EB DD C8 D8 E3 
FF DE F3 EB CD E7 E9 F0 F9 
FF FA DC EA F9 F7 DC FA FD 
FF FF FB F7 E8 FD FF F7 FB 
4F FF FF F7 FF AE 20 EB F0 BF B2 22 F1 BF B5 22 B9 20 BA 20 C0 C0 C0 80 1B 80 80 B1 21 FF 00 B0 22 B0 20 D1 21 00 DD 2C
```

#### Samples from PIC.DAT

`PIC.DAT` contains a fair amount of plaintext data. Each file within `PIC.DAT` starts with "REM", which is a keyword used to start a remark (comment) in some languages.

Each file within `PIC.DAT` corresponds to one of the game's levels; this is clear from their names in `PIC.DAT`'s table, but a level name also appears in each file (albeit broken up).

Name | Start Offset | Length (Uncompressed) | Length (Compressed)
--- | --- | --- | ---
Miss11.PIC | 1032 | 69242 | 16005

Mission 1 ("Shipyard Search") of the first set of missions ("Assault on NeoTokyo").

Bytes | Notes/ASCII
--- | ---
FF | 
52 45 4D 20 4D 49 53 53 | "REM MISS"
FF | 
31 31 20 2D 2D 20 53 68 | "11 -- Sh"
EF | EF = 239 or -17
69 70 0D 0A | "ip\r\n"
EE F1 | EE = 238 or -18; F1 = 241 or -15
41 73 73 | "Ass" (first part of "Assault")
FF | 
61 75 6C 74 20 6F 6E 20 | "ault on "
FF | 
4E 65 6F 54 6F 6B 79 6F | "NeoTokyo"
FD 2C FB F2 | FD = 253/-3, 2C = 44/44, FB = 251/-5, F2 = 242/-14
79 61 72 64 20 53 | "yard S"
7F | 7F = 127/127; delete in ASCII
65 61 72 63 68 0D 0A | "earch\r\n" 
2A 01 | "*" and start-of-heading byte in ASCII (probably not ASCII; odd characters to see)
F2 EE F1 2A 34 0F 00 03 | 
4C 61 73 74 | "Last"
7F | 127/127
20 65 64 69 74 65 64 | " edited"
F8 F0 FC 5A 0F 64 06 | Oddly long when next ASCII chunk seems to follow directly
20 31 2D 32 37 2D | " 1-27-"
E3 | 227/-29
39 37 | "97"
2E 0F 40 06 00 03 | 
43 68 61 | "Cha"
EF | EF = 239 or -17
6E 67 65 73 | "nges"
00 03 | 
46 72 65 | "Fre"
FF | 
61 6B 20 46 61 63 74 6F | "ak Facto"
FF | 
72 79 20 44 6F 6F 72 73 | "ry Doors"
3F | 63/63
20 72 65 6D 6F 76 | " remov"
56 0F D6 0F FA 68 0F 33 7B 00 20 44 42 20 4B E1 42 00 03 1B 06 B2 0B C2 03 | 
61 66 74 | "aft"
FF | 
65 72 20 74 68 65 20 47 | "er the G"
FF | 
72 6F 75 70 4F 62 6A 65 | "roupObje"
FF | 
63 74 73 20 53 74 61 74 | "cts Stat"
9F | 159/-97
65 6D 65 6E 74 | "ement"
[remaining bytes follow]

Name | Start Offset | Length (Uncompressed) | Length (Compressed)
--- | --- | --- | ---
Miss21.pic | 42248 | 212563 | 35630

Mission 1 ("Terror in Shinjuku") of the second set of missions ("Corruption").

Bytes | Notes/ASCII
--- | ---
FF | 
52 45 4D 20 4D 69 73 73 | "REM Miss"
EF | 
32 31 20 2D | "21 -"
F9 F7 | F9 = 249/-7; F7 = 247/-9
20 53 68 | " Sh"
DF | 223/-33
69 6E 44 0D 0A |  "inD\r\n" (The "D" seems out of place, might be part of compression info; D = 68/68)
EE F1 | 
43 6F | "Co" 
FF | 
72 72 75 70 74 69 6F 6E | "rruption"
FF | 
2C 20 54 65 72 72 6F 72 | ", Terror" (comma seems out of place)
F7 | 247/-9
20 69 6E | " in"
04 02 | 
6A 75 6A 75 | "juju" (assuming this is part of "Shinjuku", the second "j" should be a "k")
FB | 251/-5
0D 0A | "\r\n"
0A 03 | 
63 6F 70 69 65 | "copie"
FF | 
64 20 66 72 6F 6D 20 48 | "d from H"
FF | 
43 5F 57 6F 72 6B 5C 50 | "C_Work\P"
F3 | 243/-13
49 43 | "IC"
F8 F9 FE F4 | 
31 2D 32 33 | "1-23"
2F | 47/47
2D 39 37 20 | "-97 "
2E 05 2A 70 0F 75 01 FE 0A 03 | 
4C 61 73 74 20 52 65 | "Last Re"
F7 | 
76 69 73 | "vis"
17 00 | 
20 32 2D 35 | " 2-5"
C6 64 01 41 4A 6A 0F 7C 0A 2E 05 | 
43 68 | "Ch"
DF | 223/-33
61 6E 67 65 73 | "anges"
0A 03 | 
46 6F | "Fo"
F7 | 247/-9
67 20 72 | "g r"
CD 01 | 
20 73 68 6F | " sho"
CF | 207/-49
72 74 65 6E | "rten"
3A 01 20 00 | 
43 6C | "Cl"
3F | 63/63
6F 75 64 53 6B 79 | "oudSky"
[remaining bytes follow]

Name | Start Offset | Length (Uncompressed) | Length (Compressed)
--- | --- | --- | ---
Miss31.PIC | 184456 | 221107 | 34536

Mission 1 ("Tower Attack") of the third set of missions ("Data Space Demon").

Bytes | Notes/ASCII
--- | ---
FF |
52 45 4D 20 4D 69 73 73 | "REM Miss"
FF |
33 31 2E 50 49 43 20 2D | "31.PIC -"
FE FD F4 |
20 54 6F 77 65 72 20 | " Tower "
BF 28 | BF = 191/-65, 28 = 40/40
54 6F 6B 79 6F | "Tokyo" (not part of level name)
05 03 29 FB 0D 0A EE F1 | 
44 61 74 61 20 | "Data "
FF |
53 70 61 63 65 20 44 65 | "Space De"
EF | 239/-17
6D 6F 6E | "mon"
2C 05 04 | 
41 74 74 | "Att"
F7 | 247/-9
61 63 6B | "ack"
19 03 | 
43 6F 70 69 | "Copi"
FF | 
65 64 20 66 72 6F 6D 20 | "ed from "
7F | 
48 43 5F 57 6F 72 6B | "HC_Work"
FC F6 FE 5E 0E | 
20 20 31 2D 32 33 2D | "  1-23-"
7F | 
39 37 20 44 42 0D 0A | "97 DB\r\n"
19 03 F1 2A 84 0F 8A 02 19 03 4C 61 73 74 FF 20 52 65 76 69 73 69 6F 8D 6E 71 06 41 4A 7E 0F 90 0B 3D 04 68 DF 61 6E 67 65 73 |
[remaining bytes follow] | 
