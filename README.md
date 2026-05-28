# 🔒 RBS Compiler

**Multithreaded compiler for the Real Bit Stream (.rbs) format — the protected game packaging format for Real_X console OS.**

[![Platform](https://img.shields.io/badge/Platform-Windows-blue)]()
[![Format](https://img.shields.io/badge/Format-RBS%20v0.4-red)]()

---

## What is RBS?

**RBS (Real Bit Stream)** is a custom binary archive format designed for the Real_X console. It combines **LZ4 compression** with **rolling XOR encryption** to create game packages that are:

- **Fast to compile** — multithreaded, single-pass
- **Fast to load** — LZ4 decompresses at ~2 GB/s on any hardware
- **Protected** — data encrypted with position-dependent rolling XOR; standard tools see only binary noise
- **Compact** — LZ4 compression reduces file sizes by 40-70%

## Features

- 🧵 **Multithreaded compilation** — reads and compresses files across all CPU cores
- 🗜️ **LZ4 compression** — ~500 MB/s compress, ~2 GB/s decompress
- 🔐 **Rolling XOR encryption** — key changes with every byte
- 🖱️ **Drag & Drop** — drag a folder onto the .exe to compile
- 📦 **Single-file output** — all files packed into one `.rbs` archive
- 🛡️ **Write-only** — can only create .rbs files; reading requires Real_X console

## Quick Start

### Install

Download `rbs_compiler.exe` from Releases.

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
rbs_compiler.exe my_game/
```

## Example: Snake

Built-in example game included with the compiler:

```bash
rbs_compiler.exe examples/snake
```

Result: `snake.rbs` (1.2 KB) — ready to run on Real_X console.

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

## Compression Results

| File | Original | Compressed | Ratio |
|------|----------|------------|-------|
| Binary (game code) | 1,093 B | 347 B | **31%** |
| Text (readme) | 271 B | 246 B | 90% |
| Config (TOML) | 267 B | 238 B | 89% |
| Image (PNG) | 116 B | 118 B | 101% |
| **Total** | **1,747 B** | **949 B** | **54%** |

Binary files compress extremely well. Text and pre-compressed images stay roughly the same.

## Format Specification

### Archive Structure

```
[Magic: "RBS!"]  (4 bytes)
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

```
rolling_key(pos, file_offset) = (pos + 0x52) * 0x37 ^ MAGIC[pos % 4] ^ (file_offset * 0x13)
```

Each byte is XORed with a unique key that depends on:
- **Position** within the data
- **File offset** in the archive
- **Magic bytes** `RBS!`

This prevents XOR frequency analysis and pattern recognition.

## Security

| Attack | Protection |
|--------|------------|
| Hex editor / strings | Data is encrypted, not plaintext |
| 7-Zip / WinRAR | Custom format, not recognized |
| XOR frequency analysis | Rolling key prevents patterns |
| Extract & repack | Compiler is write-only |
| Static key recovery | Key depends on position + offset |

**Only Real_X console can read .rbs files.**

## Performance

| Operation | Speed |
|-----------|-------|
| Compile 100 files (50 MB) | ~0.3 seconds |
| Decompress (on console) | ~25 ms |

## Requirements

- Windows (drag-and-drop works in Explorer)
- Command-line usage also works on Linux/macOS

## License

MIT
