# Huffman Coding Compression (C++)

A command-line file compressor and decompressor built in modern C++ using **Huffman coding**, a lossless, prefix-based entropy encoding algorithm. It works on any file type (text, source code, logs, binaries) and restores the original data byte-for-byte.

![Language](https://img.shields.io/badge/language-C%2B%2B17-blue)
![Build](https://img.shields.io/badge/build-CMake-success)
![License](https://img.shields.io/badge/license-MIT-green)

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [How It Works](#how-it-works)
- [Features](#features)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Usage](#usage)
- [Compressed File Format](#compressed-file-format)
- [Example](#example)
- [Complexity](#complexity)
- [Testing](#testing)
- [Limitations](#limitations)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Problem Statement

Most files store every symbol (byte) using a fixed number of bits (8), even when some symbols appear far more often than others. This wastes space. Huffman coding addresses this by assigning **shorter bit codes to frequent symbols** and **longer codes to rare ones**, reducing total file size without losing any information.

## How It Works

**Compression**

1. **Read** the input file and count the frequency of each byte (0–255).
2. **Build** a min-heap (priority queue) of leaf nodes, one per distinct byte.
3. **Merge** the two lowest-frequency nodes repeatedly into a parent node until a single root remains. This produces the Huffman tree.
4. **Generate codes** by traversing the tree (left edge = `0`, right edge = `1`). Each leaf's path is its prefix-free code.
5. **Encode** the input by replacing each byte with its code and packing the bits into bytes.
6. **Write** a header (metadata + frequency table) followed by the packed bitstream.

**Decompression**

1. **Read** the header and rebuild the identical Huffman tree from the stored frequency table.
2. **Decode** the bitstream by walking the tree from the root, emitting a byte each time a leaf is reached.
3. **Stop** after the original number of bytes has been written, ignoring padding bits.

Because no code is a prefix of another, decoding is unambiguous.

## Features

- Lossless compression and decompression of any file type
- Handles edge cases: empty files, single-symbol files, and large files
- Deterministic tree construction, so compressor and decompressor always agree
- Bit-level I/O with correct padding handling
- Magic number and size validation to detect invalid or corrupted input
- Reports compression ratio after each run
- Clean, modular code with no external dependencies

## Project Structure

```
huffman-compression/
├── CMakeLists.txt
├── README.md
├── LICENSE
├── include/
│   ├── huffman_tree.hpp      # Node definition, tree construction, code generation
│   ├── bit_io.hpp            # BitWriter / BitReader utilities
│   └── codec.hpp             # compress() / decompress() public API
├── src/
│   ├── main.cpp              # CLI entry point
│   ├── huffman_tree.cpp
│   ├── bit_io.cpp
│   └── codec.cpp
├── tests/
│   ├── test_roundtrip.cpp    # compress -> decompress -> compare
│   └── data/                 # sample files for testing
└── docs/
    └── format.md             # (optional) detailed file format notes
```

> Adjust this tree to match your actual repository layout.

## Getting Started

### Prerequisites

- A C++17 (or newer) compiler: GCC 9+, Clang 10+, or MSVC 2019+
- CMake 3.15+ (optional; a plain `g++` command also works)

### Build with CMake

```bash
git clone https://github.com/<your-username>/huffman-compression.git
cd huffman-compression

cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

The executable will be at `build/huffman`.

### Build without CMake

```bash
g++ -std=c++17 -O2 -Iinclude src/*.cpp -o huffman
```

## Usage

```bash
# Compress
./huffman compress <input_file> <output_file>

# Decompress
./huffman decompress <input_file> <output_file>
```

| Command      | Description                                   |
|--------------|-----------------------------------------------|
| `compress`   | Compresses `<input_file>` into `<output_file>`   |
| `decompress` | Restores the original file from a compressed one |

### Verify integrity

```bash
./huffman compress sample.txt sample.huf
./huffman decompress sample.huf restored.txt
cmp sample.txt restored.txt && echo "Round-trip OK"
```

## Compressed File Format

| Section            | Size                | Purpose                                              |
|--------------------|---------------------|------------------------------------------------------|
| Magic number       | 4 bytes             | Identifies the file as this format (e.g. `HUFF`)     |
| Original size      | 8 bytes             | Number of bytes in the uncompressed data             |
| Symbol count       | 2 bytes             | Number of distinct bytes stored in the table         |
| Frequency table    | 5 bytes per symbol  | 1-byte symbol + 4-byte frequency                     |
| Encoded bitstream  | Variable            | Packed Huffman codes; final byte is zero-padded      |

Storing the original size lets the decoder ignore padding bits and detect truncated files.

> Update this table if your header layout differs.

## Example

```text
$ ./huffman compress war_and_peace.txt war_and_peace.huf
Original size   : 3,359,372 bytes
Compressed size : 1,921,408 bytes
Ratio           : 57.2%  (42.8% saved)

$ ./huffman decompress war_and_peace.huf restored.txt
Decompressed 3,359,372 bytes successfully.
```

*Figures above are illustrative. Replace them with results from your own benchmarks.*

## Complexity

Let *n* be the input size in bytes and *k* the number of distinct symbols (at most 256).

| Phase                 | Time              | Space   |
|-----------------------|-------------------|---------|
| Frequency counting    | O(n)              | O(k)    |
| Tree construction     | O(k log k)        | O(k)    |
| Encoding              | O(n)              | O(n)    |
| Decoding              | O(n)              | O(n)    |

Since *k* is bounded by 256, tree construction is effectively constant time. Overall runtime is linear in file size.

## Testing

```bash
cd build
ctest --output-on-failure
```

The test suite covers:

- Round-trip correctness on text and binary files
- Empty file
- Single-byte and single-repeated-symbol files (tree with one node)
- Files where all 256 byte values appear
- Large files (multi-MB)
- Corrupted or truncated input (expects a clean error, not a crash)

## Limitations

- **Small files may grow.** The frequency table adds overhead, so files under a few hundred bytes can end up larger after compression.
- **Already-compressed data** (ZIP, JPEG, MP4) gains little or nothing, since its byte distribution is near uniform.
- **Whole-file model.** The entire file is read into memory and a single tree is used. Very large files may need a streaming or block-based approach.
- Compression ratio is lower than modern algorithms (DEFLATE, Zstandard, LZMA) because Huffman coding alone does not exploit repeated sequences.

## Roadmap

- [ ] Canonical Huffman codes to shrink the header
- [ ] Block-based / streaming compression for large files
- [ ] Multi-file and directory support
- [ ] Checksum (CRC32) for integrity verification
- [ ] Benchmark suite against gzip and zstd
- [ ] Multithreaded frequency counting

## Contributing

Contributions are welcome.

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit your changes with clear messages
4. Ensure all tests pass
5. Open a pull request describing the problem and your solution

Please follow the existing code style and add tests for new behavior.

## License

Distributed under the MIT License. See [`LICENSE`](LICENSE) for details.

## Acknowledgments

- D. A. Huffman, *"A Method for the Construction of Minimum-Redundancy Codes"* (1952)
- *Introduction to Algorithms* (CLRS), Chapter 16: Greedy Algorithms
