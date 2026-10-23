# Huffman Compression (C++)

A lightweight command-line tool that compresses and decompresses files using **Huffman coding**, a lossless, prefix-free entropy encoding algorithm. The entire implementation lives in a single C++ source file with no external dependencies.

![Language](https://img.shields.io/badge/language-C%2B%2B17-blue)
![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20Windows-lightgrey)

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Features](#features)
- [How It Works](#how-it-works)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Usage](#usage)
- [Compressed File Format (`.abiz`)](#compressed-file-format-abiz)
- [Code Overview](#code-overview)
- [Complexity](#complexity)
- [Known Limitations](#known-limitations)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [Author](#author)

---

## Problem Statement

Standard file storage uses a fixed 8 bits per byte, regardless of how often each byte value actually occurs. Huffman coding removes that waste by assigning **shorter bit patterns to frequent bytes** and **longer patterns to rare ones**. The result is a smaller file that decompresses back to the original, byte for byte.

## Features

- Lossless compression and decompression of any file type (text, source code, logs, binaries)
- Custom `.abiz` output format with a self-contained header, so no external dictionary is needed
- Correct handling of single-symbol files (assigned a 1-bit code)
- Bit-level packing with padding tracked in the header
- Pre-compression size estimate printed before encoding begins
- Live progress percentage and timing for both operations
- Single source file, easy to read, build, and extend

## How It Works

### Compression (`-c`)

1. **Measure** the input file size (64-bit safe).
2. **Count** the frequency of every byte value (0-255).
3. **Build the Huffman tree.** Leaf nodes are sorted by frequency, and the two lowest-count nodes are repeatedly merged into a parent until one root remains.
4. **Generate codes** by walking the tree (left edge = `0`, right edge = `1`) and store them in a lookup table. The total encoded size in bits is calculated at the same time.
5. **Compute padding** needed to round the bitstream up to a whole byte.
6. **Write the header** (symbol table and padding), followed by the packed bitstream (most significant bit first).

### Decompression (`-dc`)

1. **Read the header** and rebuild the Huffman tree from the stored codes.
2. **Walk the bitstream** bit by bit from the root of the tree. Each time a leaf is reached, its byte is written out and traversal restarts at the root.
3. **Stop at the final byte** using the stored padding value so the trailing filler bits are ignored.

Because no code is a prefix of another, decoding is unambiguous.

## Project Structure

```
.
├── .gitignore                 # Excludes executables and .vscode
├── README.md
└── huffman-compression.cpp    # Complete implementation (compressor + decompressor + CLI)
```

## Getting Started

### Prerequisites

- A C++17 compatible compiler (GCC 7+, Clang 5+, or MinGW-w64). C++17 is required because the decompressor uses structured bindings.

### Clone and Build

```bash
git clone https://github.com/SumitDutta007/<repo-name>.git
cd <repo-name>

g++ -std=c++17 -O2 huffman-compression.cpp -o huffman
```

On Windows (MinGW), the output will be `huffman.exe`.

> **Note:** The code uses `ftello64` to read file sizes, which is available on Linux and MinGW. On macOS, where `ftello64` is not provided, replace it with `ftello`.

## Usage

```bash
# Compress
./huffman -c <file_to_compress>

# Decompress
./huffman -dc <file_to_decompress>.abiz
```

| Flag  | Description                                                    |
|-------|----------------------------------------------------------------|
| `-c`  | Compress the given file. Output is `<filename>.abiz`           |
| `-dc` | Decompress a `.abiz` file. Output is `output<filename>` (the `.abiz` extension is removed) |

### Naming Convention

| Command                     | Output file       |
|-----------------------------|-------------------|
| `./huffman -c notes.txt`    | `notes.txt.abiz`  |
| `./huffman -dc notes.txt.abiz` | `outputnotes.txt` |

The `output` prefix prevents the decompressed file from overwriting your original.

### Verify the Round Trip

```bash
./huffman -c notes.txt
./huffman -dc notes.txt.abiz
cmp notes.txt outputnotes.txt && echo "Round-trip OK"
```

### Sample Console Output

```text
notes.txt
Original File: 1048576 bytes
Compressed File Size (without header): 612345 bytes
Padding size: 3
100% completed
Compression Time: 0.42s
```

*Values are illustrative. Your results depend on the input data.*

> **Tip:** Run the tool from the directory containing the file. The decompressor prepends `output` to the full path you pass in, so paths with folders (e.g. `data/notes.txt.abiz`) will not produce the expected output location.

## Compressed File Format (`.abiz`)

```
+-------------------------------+
| Unique symbol count - 1       |  1 byte  (0 means 1 symbol, 255 means 256)
+-------------------------------+
| For each unique symbol:       |
|   Symbol                      |  1 byte
|   Code length (L)             |  1 byte
|   Huffman code                |  L bytes, stored as ASCII '0' / '1'
+-------------------------------+
| Padding bits                  |  1 byte  (unused bits in the last data byte)
+-------------------------------+
| Encoded bitstream             |  N bytes, MSB first, zero-padded at the end
+-------------------------------+
```

- **Best-case header:** about 5 bytes (file contains a single distinct byte value).
- **Worst-case header:** roughly 32 KB, which only occurs with a highly skewed tree where code lengths approach 255 bits.
- Codes are stored as readable characters rather than packed bits. This keeps the header simple to parse at the cost of extra size on files with many distinct symbols.

## Code Overview

| Component                      | Responsibility                                                         |
|--------------------------------|------------------------------------------------------------------------|
| `Huffman::Node`                | Tree node holding a byte, its count, and left/right children           |
| `Huffman::HuffmanValue[256]`   | Lookup table mapping each byte to its binary code string               |
| `Utility::get_file_size`       | Returns the file size as a 64-bit value                                |
| `Utility::Inorder`             | Debug helper that prints each character with its count and code        |
| `CompressUtility::parse_file`  | Builds the byte-frequency map                                          |
| `CompressUtility::generate_huffman_tree` | Constructs the tree by repeatedly merging the lowest-count nodes |
| `CompressUtility::store_huffman_value`   | Fills the code table and returns the encoded size in bits    |
| `CompressUtility::generate_header`       | Serializes the symbol table and padding                      |
| `CompressUtility::compress`    | Writes the header and the bit-packed payload                           |
| `DecompressUtility::decode_header`       | Reads the header and rebuilds the tree                       |
| `DecompressUtility::decompress` | Decodes the bitstream back to the original bytes                      |

## Complexity

Let *n* be the input size in bytes and *k* the number of distinct byte values (at most 256).

| Phase                | Time                          | Space |
|----------------------|-------------------------------|-------|
| Frequency counting   | O(n)                          | O(k)  |
| Tree construction    | O(k² log k) worst case        | O(k)  |
| Encoding             | O(n)                          | O(k)  |
| Decoding             | O(total encoded bits)         | O(k)  |

Because *k* is capped at 256, tree construction is negligible in practice and runtime is dominated by file I/O.

## Known Limitations

- **Empty files are not supported.** An empty input leaves the tree builder with no nodes.
- **Small files can grow.** The header adds overhead, so tiny files may end up larger than the original.
- **Already-compressed data** (ZIP, JPEG, MP4, etc.) will see little to no reduction because its byte distribution is nearly uniform.
- **Path handling in decompression.** The `output` prefix is added to the entire path argument, so run the tool from the file's directory.
- **No integrity checks.** There is no magic number or checksum, so a corrupted or non-`.abiz` file is not detected gracefully.
- **Unbuffered per-byte I/O** (`fgetc` / `fputc`) limits throughput on very large files.
- **Tree nodes are not freed.** This is harmless for a short-lived CLI process but worth addressing if the code is reused as a library.

## Roadmap

- [ ] Replace the sorted vector with `std::priority_queue` for tree construction
- [ ] Add a magic number and CRC32 checksum to detect invalid or corrupted files
- [ ] Support empty files
- [ ] Pack codes in the header as bits (or use canonical Huffman codes) to reduce header size
- [ ] Use buffered block I/O for faster processing of large files
- [ ] Let the user choose the output file name and path
- [ ] Release tree memory and split the code into headers and source files
- [ ] Add automated round-trip tests and a benchmark against gzip

## Contributing

Contributions and suggestions are welcome.

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit your changes with a clear message
4. Push the branch and open a pull request describing the problem and the fix

## Author

**Sumit Dutta**: [@SumitDutta007](https://github.com/SumitDutta007)
