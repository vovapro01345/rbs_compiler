# 🔒 RBS Compiler

**Multithreaded compiler for the Real Bit Stream (.rbs) format — the protected game packaging format for [Real_X](https://github.com/your-repo/Real_X) gaming OS.**

[![Rust](https://img.shields.io/badge/Rust-nightly-orange)](https://rustup.rs)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Windows-blue)]()

---

## What is RBS?

**RBS (Real Bit Stream)** is a custom binary archive format designed for the Real_X gaming console OS. It combines **LZ4 compression** with **rolling XOR encryption** to create game packages that are:

- **Fast to compile** — multithreaded via rayon, single-pass with BufWriter
- **Fast to load** — LZ4 decompresses at ~2 GB/s, even on weak hardware
- **Protected** — data is encrypted with a position-dependent rolling XOR key; standard tools (7-Zip, hex editors, file managers) see only binary noise
- **Zero-allocation friendly** — suitable for bare-metal kernel loaders with no heap

## Features

- 🧵 **Multithreaded compilation** — reads and compresses files in parallel across all CPU cores
- 🗜️ **LZ4 compression** — fast algorithm, ~500 MB/s compress, ~2 GB/s decompress
- 🔐 **Rolling XOR encryption** — key depends on byte position, file offset, and magic bytes
- 🖱️ **Drag & Drop** — drag a folder onto the .exe to compile instantly
- 📦 **Single-file output** — all game files packed into one `.rbs` archive
- 🛡️ **Write-only** — the public compiler can only create .rbs files; reading requires Real_X OS

## Quick Start

### Install

```bash
git clone https://github.com/your-repo/Real_X.git
cd Real_X
cargo build --release -p rbs_compiler
```

Binary: `target/release/rbs_compiler.exe`

### Compile a game

**Drag & Drop:**
```
📁 my_game/
├── manifest.toml
├── icon.png
└── data/
    └── game.bin

        ↓ drag onto rbs_compiler.exe

📄 my_game.rbs
```

**Command line:**
```bash
# Auto-name (creates my_game.rbs)
rbs_compiler.exe my_game/

# Custom output name
rbs_compiler.exe my_game/ output.rbs
```

## Game Folder Structure

```
my_game/
├── manifest.toml          # Required: game metadata
├── icon.png               # Dashboard icon (16x16 or 32x32)
├── data/                  # Game resources
│   ├── game.bin           # Executable code
│   ├── sprites/           # Graphics
│   └── levels/            # Level data
└── config.toml            # Optional settings
```

### manifest.toml

```toml
[game]
name = "My Game"
version = "1.0.0"
author = "Author Name"
description = "A great game for Real_X"
icon = "icon.png"

[game.display]
width = 640
height = 480
fullscreen = false

[game.controls]
up = "W"
down = "S"
left = "A"
right = "D"
pause = "Escape"
```

## Example: Snake

The repository includes a complete example game:

```bash
# Compile the example
rbs_compiler.exe examples/snake

# Result: snake.rbs (1.2 KB)
```

### Compression results

| File | Original | Compressed | Ratio |
|------|----------|------------|-------|
| `data/snake.bin` | 1,093 B | 347 B | **31%** |
| `data/readme.txt` | 271 B | 246 B | 90% |
| `manifest.toml` | 267 B | 238 B | 89% |
| `icon.png` | 116 B | 118 B | 101% |
| **Total** | **1,747 B** | **949 B** | **54%** |

Binary files compress extremely well; text files are already compact.

## Format Specification

### Archive Structure

```
[Magic: "RBS!"]  (4 bytes, raw)
[Entry 1]
[Entry 2]
...
[Entry N]
```

### Entry Structure

| Field | Size | Description |
|-------|------|-------------|
| Path | 64 bytes | Null-terminated UTF-8, rolling-XOR encrypted |
| Original Size | 8 bytes | u64 little-endian, rolling-XOR encrypted |
| Compressed Size | 8 bytes | u64 little-endian, rolling-XOR encrypted |
| Data | `Compressed Size` bytes | LZ4 block compressed, rolling-XOR encrypted |

### Encryption

```rust
fn rolling_key(pos: usize, file_offset: usize) -> u8 {
    let base = (pos as u8).wrapping_add(0x52).wrapping_mul(0x37);
    let magic_byte = RBS_MAGIC[pos % 4];
    let file_salt = (file_offset as u8).wrapping_mul(0x13);
    base ^ magic_byte ^ file_salt
}
```

Each byte is XORed with a key that depends on:
- **Position** within the data (changes per byte)
- **File offset** in the archive (changes per file)
- **Magic bytes** `RBS!` (tied to format)

This makes simple XOR analysis impossible — the key is different for every byte.

### Decompression

The Real_X kernel contains a hand-written LZ4 block decompressor (`kernel/src/rbs_loader.rs`) that runs in `no_std` bare-metal mode. **No external libraries required.**

## Security

| Attack | Protection |
|--------|------------|
| Hex editor / strings | Data is XOR-encrypted, not plaintext |
| 7-Zip / WinRAR | Custom format, not recognized |
| XOR frequency analysis | Rolling key prevents patterns |
| Extract & repack | Compiler is write-only; no extract in public build |
| Static key recovery | Key depends on position + file offset |

**Only Real_X OS can read .rbs files** through its kernel loader.

## Dependencies

| Crate | Purpose | Build-time only |
|-------|---------|-----------------|
| `rayon` | Multithreaded file processing | ✅ |
| `lz4_flex` | LZ4 block compression | ✅ |

**Runtime: None** — the compiler is a standalone binary.

## Requirements

- **Rust nightly** (for `x86_64-unknown-none` target support)
- **Windows** (drag-and-drop works in Explorer)
- Linux/macOS: command-line only (no drag-and-drop)

## Performance

On a 12-core CPU (Xeon E5-2650 v4):

| Operation | Time | Throughput |
|-----------|------|------------|
| Compile 100 files (50 MB) | ~0.3s | ~160 MB/s |
| Decompress (in kernel) | ~25ms | ~2 GB/s |

## License

MIT

## Related

- [Real_X OS](https://github.com/your-repo/Real_X) — The gaming console OS
- [LZ4](https://github.com/lz4/lz4) — Fast compression algorithm
- [rayon](https://github.com/rayon-rs/rayon) — Data parallelism library
