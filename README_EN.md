[English](./README_EN.md) | [中文](./README.md)

# QR Code Generator in Minecraft

A QR code generator implemented in Minecraft Bedrock Edition using `mcfunction`, scoreboards, entities, and blocks. Input encoding, Reed–Solomon error correction, block splitting and interleaving, matrix placement, and mask XOR all run inside the game.

The project organizes computation in an assembly-like way: **scoreboards act as registers, entities act as pointers, `qr_prg` holds execution state, and blocks store binary data.** The world makes it possible to observe data being written, moved, and processed.

## Supported features

- Minecraft Bedrock Edition; the behavior pack declares **1.21.0** as its minimum engine version.
- QR code **versions 1–40 with error correction level L**.
- Byte-mode encoding, with ASCII character input through the supplied input panel.
- Versions 1 and 2 directly generate **8 mask results** per run; versions 3–40 generate **1 result** per run.
- **The accompanying world is required for complete operation.** It also supplies input interaction, command block scheduling, and additional structure data.

## How to use

1. Download and extract a release from the project's [Releases page](https://github.com/baby20162016/QR_Code-Generator-in-Minecraft/releases).
2. Import `QR_Code Generator.mcpack` and `QR_Code Generator.mcworld` into Minecraft Bedrock Edition.
3. Enter the supplied world, use the input panel buttons to enter content, select a QR code version, and press the generate button.
4. Wait for encoding, error correction, and matrix placement to finish. Versions 1 and 2 produce 8 QR codes together; later versions produce 1.

`structures/main.mcstructure` is the input panel. Importing only the behavior pack and running `/function QR/main` does not replace the complete environment supplied by the world.

## How it works

### 1. A computation model built from commands

| Minecraft mechanism | Role in this project |
| --- | --- |
| Scoreboard scores | Store values, counters, and intermediate results, like registers |
| `qr_prg` | Holds execution state; `execute ... scores=...` selects commands to execute |
| Armor stand positions | Point to the blocks being read or written; moving an entity moves its pointer |
| Entity names, IDs, and tags | Distinguish the main process, input characters, byte operations, and read/write pointers |
| Black and white concrete | Store bits: black is 1, white is 0 |
| `structure save/load` | Copy, back up, and move block data for shifts and workspace changes |
| Route marker blocks | Encode movement directions and jump distances for the matrix write pointer |

“Assembly-like” describes the roles of these underlying mechanisms. Functions advance through execution states; changing state can allow later commands in the same function call to execute the next stage. Some functions are called repeatedly to perform several steps within one scheduling cycle.

### 2. From characters to a bitstream

`ASCII.mcfunction` creates armor stands representing input characters, stores their order in `qr_uid`, and stores their values in `qr_encode`. During generation, `data_code.mcfunction` writes the byte-mode indicator, character count, and character data. The count field uses 8 bits for versions 1–9 and 16 bits for versions 10–40.

`encode.mcfunction` extracts the least significant bit with `% 2`, advances with `/ 2`, and writes concrete from right to left. This produces a bitstream whose most significant bits come first when read from left to right. Each data row holds 64 bits; the pointer moves up one layer at the row boundary.

After input encoding, the stream receives terminator bits and byte alignment. `pad.mcfunction` alternates `0xEC` and `0x11` to fill the selected version's data codeword capacity.

### 3. Error correction inside the game

Reed–Solomon error correction requires arithmetic in GF(256). Generator polynomial coefficients and logarithm/antilogarithm mappings are predefined in functions; error correction for the actual input is computed at runtime:

1. `decode.mcfunction` reads 8 blocks and reconstructs a byte using weights `1, 2, 4, 8, 16, 32, 64, 128`.
2. `GF_2.mcfunction` maps a nonzero byte to its exponent. Multiplication by a generator polynomial coefficient uses exponent addition modulo 255.
3. `GF_1.mcfunction` maps the resulting exponent back to a byte, and `encode_sub.mcfunction` writes that byte as blocks.
4. `xor.mcfunction` performs bitwise XOR between the data and product areas to obtain the next remainder. Data is moved for the next iteration; `sup.mcfunction` removes leading zero bytes.

XOR itself is implemented with blocks: equal bits produce white concrete, and different bits produce black concrete. Orientations of entities on either side and successive local-coordinate offsets expand execution across multiple positions in a row. Repeated calls advance vertically through the rows.

### 4. Splitting and interleaving

Versions 1–5 use one data block; versions 6–40 split data according to the selected version. `config_split.mcfunction` supplies block lengths, while `split.mcfunction` moves data byte by byte. The main process computes error correction for each block and backs up and restores the generator polynomial coefficients.

`read_high.mcfunction` interleaves bytes across blocks, reading data codewords first and error correction codewords afterward. For versions with two groups of different data block lengths, additional pointer markers handle the remaining bytes from the longer blocks. Versions 1–5 use `read_low.mcfunction` for sequential reading.

### 5. Matrix construction, placement, and masking

The QR matrix side length is `4 × version + 17`. Versions 1–12 load their framework and placement routes from `qr_mode_*` structures. Versions 13–40 use `mode_summon.mcfunction` to generate the framework and routes, together with finder patterns, alignment patterns, and version information structures.

During placement, the `qr_fill` pointer reads colored wool and other marker blocks in the route layer. These markers direct turns and jumps around function patterns while the interleaved bitstream is written into the matrix.

Versions 1 and 2 copy the placed data into 8 workspaces and apply 8 mask structures. Versions 3–12 use one mask structure. Versions 13–40 use `matrix_summon.mcfunction` to generate a checkerboard mask from the parity of the coordinate sum. Finally, `matrix.mcfunction` XORs the data and mask layers while preserving function patterns to produce the QR code.

## Source guide

Start with `QR.mcfunction` to follow state transitions, then read the encoding, error correction, and placement functions.

| File or directory | Purpose |
| --- | --- |
| [manifest.json](./manifest.json) | Behavior pack metadata and minimum engine version |
| [functions/QR/main.mcfunction](./functions/QR/main.mcfunction) | Entry point calling QR/QR |
| [functions/QR/QR.mcfunction](./functions/QR/QR.mcfunction) | Main state flow: initialization, error correction, block switching, and placement |
| [functions/QR/ASCII.mcfunction](./functions/QR/ASCII.mcfunction) | Character input, IDs, deletion/reset, and generation trigger |
| [functions/QR/version.mcfunction](./functions/QR/version.mcfunction) | Capacity, error correction length, and block count for versions 1–40 |
| [functions/QR/data_code.mcfunction](./functions/QR/data_code.mcfunction) | Character count, character data, termination, and splitting stages |
| [functions/QR/encode.mcfunction](./functions/QR/encode.mcfunction) | Value-to-block encoding; encode_sub writes bytes for error correction |
| [functions/QR/encode_sub.mcfunction](./functions/QR/encode_sub.mcfunction) | Parallel byte encoding during error correction |
| [functions/QR/decode.mcfunction](./functions/QR/decode.mcfunction) | Reconstruct a byte from 8 blocks |
| [functions/QR/pad.mcfunction](./functions/QR/pad.mcfunction) | Alternating padding codewords |
| [functions/QR/GF/](./functions/QR/GF/) | Lookup mappings between GF(256) exponents and byte values |
| [functions/QR/generator/](./functions/QR/generator/) | Predefined generator coefficients grouped by error correction length |
| [functions/QR/xor.mcfunction](./functions/QR/xor.mcfunction) | Block XOR |
| [functions/QR/sup.mcfunction](./functions/QR/sup.mcfunction) | Remove leading zero bytes |
| [functions/QR/split.mcfunction](./functions/QR/split.mcfunction) | Data codeword splitting and movement |
| [functions/QR/read_low.mcfunction](./functions/QR/read_low.mcfunction) | Sequential single-block reading |
| [functions/QR/read_high.mcfunction](./functions/QR/read_high.mcfunction) | Interleaved multi-block reading |
| [functions/QR/summon.mcfunction](./functions/QR/summon.mcfunction) | Placement pointer, route interpretation, mask loading, and final operation scheduling |
| [functions/QR/main_sub.mcfunction](./functions/QR/main_sub.mcfunction) | Repeated calls to QR/summon to advance placement and final operations |
| [functions/QR/mode_summon.mcfunction](./functions/QR/mode_summon.mcfunction) | Framework, alignment patterns, and placement routes for versions 13–40 |
| [functions/QR/matrix_summon.mcfunction](./functions/QR/matrix_summon.mcfunction) | Checkerboard mask generation |
| [functions/QR/matrix.mcfunction](./functions/QR/matrix.mcfunction) | Final XOR of data and mask layers |
| [functions/QR/config/](./functions/QR/config/) | Generator selection, workspace zero filling, framework selection, block lengths, and alignment positions |
| [functions/math/NUM.mcfunction](./functions/math/NUM.mcfunction) | Scoreboard constants; the directory also contains other math functions |
| [structures/](./structures/) | Input panel main, frameworks qr_mode_*, masks qr_matrix*, version information qr_via_*, finder and alignment patterns |

Command blocks and structure data in the world are also part of maintenance. For example, `a`, `b`, and `c`, referenced by the main process, are absent from the repository's `structures/` directory. Understanding scheduling and workspace layout requires inspecting the accompanying world as well.

## License

[MIT License](./LICENSE) · By Baby_2016
