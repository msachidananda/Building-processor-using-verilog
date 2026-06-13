# 🔧 Custom 32-bit RISC Processor — Verilog HDL

<div align="center">

![Verilog](https://img.shields.io/badge/Language-Verilog%20HDL-blue?style=for-the-badge&logo=v&logoColor=white)
![Architecture](https://img.shields.io/badge/Architecture-32--bit%20RISC-green?style=for-the-badge)
![ISA](https://img.shields.io/badge/ISA-27%20Instructions-orange?style=for-the-badge)
![Status](https://img.shields.io/badge/Simulation-Verified%20✓-brightgreen?style=for-the-badge)
![Domain](https://img.shields.io/badge/Domain-VLSI%20%2F%20RTL%20Design-purple?style=for-the-badge)

**A fully custom 32-bit RISC processor designed from scratch in Verilog HDL,  
featuring a 27-instruction ISA, flag-based conditional branching, and RTL simulation verification.**

</div>

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [Instruction Register (IR) Bit Encoding](#-instruction-register-ir-bit-encoding)
- [Instruction Set Architecture (ISA)](#-instruction-set-architecture-isa)
- [Addressing Modes](#-addressing-modes)
- [Test Program — Multiply via Repeated Addition](#-test-program--multiply-via-repeated-addition)
- [Simulation Results](#-simulation-results)
- [File Structure](#-file-structure)
- [How to Run](#-how-to-run)

---

## 🧠 Overview

This project implements a custom **32-bit RISC (Reduced Instruction Set Computer) processor** in Verilog HDL, designed entirely from first principles. The processor supports:

- A **27-instruction ISA** spanning arithmetic, logical, memory/IO, branch, and control operations
- A **dual-mode operand encoding** — register-register or register-immediate, selected by a single control bit in the IR
- **9 conditional jump variants** based on ALU status flags (carry, sign, zero, overflow — and their complements)
- A **register file** with writeback, verified against waveform simulation output

The design was verified by loading a multiply-by-repeated-addition program into instruction memory and confirming the output in simulation.

---

## 🏗 Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        RISC Processor Top                           │
│                                                                     │
│   ┌──────────┐    ┌───────────┐    ┌──────────┐    ┌───────────┐  │
│   │  Program │───▶│ Instruction│───▶│  Decode  │───▶│    ALU    │  │
│   │  Counter │    │   Memory  │    │  (IR)    │    │           │  │
│   └──────────┘    └───────────┘    └──────────┘    └─────┬─────┘  │
│        ▲                                  │               │        │
│        │                                  ▼               ▼        │
│        │                          ┌──────────────┐  ┌──────────┐  │
│        │◀─── Branch / JUMP ───────│ Control Unit │  │ Register │  │
│                                   │ (Flag Logic) │  │  File    │  │
│                                   └──────────────┘  └──────────┘  │
│                                          │                         │
│                                          ▼                         │
│                                   ┌──────────┐                     │
│                                   │  Data    │                     │
│                                   │  Memory  │                     │
│                                   └──────────┘                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

| Design Choice | Detail |
|---|---|
| **Word width** | 32-bit instructions, 16-bit data paths |
| **Register file** | 32 general-purpose registers (5-bit addressing) |
| **Instruction memory** | Initialized via `.mem` file (`inst_data.mem`) |
| **Operand selection** | Single `imm_mode` bit in IR[16] — no separate instruction formats |
| **Status flags** | Carry, Sign, Zero, Overflow — each with a conditional jump variant |
| **Special register** | `SGPR` — status/general-purpose register, accessible via `movsgpr` |

---

## 📐 Instruction Register (IR) Bit Encoding

Every instruction is encoded as a **single 32-bit word**. The IR is sliced into named fields using Verilog `` `define `` macros:

```verilog
`define oper_type   IR[31:27]   // 5-bit opcode field
`define rdst        IR[26:22]   // destination register
`define rsrc1       IR[21:17]   // source register 1
`define imm_mode    IR[16]      // 0 = reg-reg, 1 = immediate
`define rsrc2       IR[15:11]   // source register 2 (when imm_mode = 0)
`define isrc        IR[15:0]    // 16-bit immediate value (when imm_mode = 1)
```

### Bit Field Layout

```
 31      27 26     22 21     17  16  15     11 10          0
┌──────────┬─────────┬─────────┬───┬─────────┬────────────┐
│oper_type │  rdst   │  rsrc1  │imm│  rsrc2  │  (unused)  │
│ [31:27]  │ [26:22] │ [21:17] │[16]│ [15:11] │            │
│  5 bits  │  5 bits │  5 bits │ 1 │  5 bits │            │
└──────────┴─────────┴─────────┴───┴─────────┴────────────┘

                                 ↕  imm_mode = 1 → isrc overlaps [15:0]

 31      27 26     22 21     17  16  15                    0
┌──────────┬─────────┬─────────┬───┬────────────────────────┐
│oper_type │  rdst   │  rsrc1  │ 1 │        isrc [15:0]      │
│          │         │         │   │    16-bit immediate      │
└──────────┴─────────┴─────────┴───┴────────────────────────┘
```

> **Key insight:** The `imm_mode` bit at IR[16] acts as a multiplexer selector in the decode stage — the same 32-bit encoding serves both register-register and register-immediate operations without needing separate instruction formats. This is a classical RISC encoding strategy.

---

## 📖 Instruction Set Architecture (ISA)

The processor supports **27 instructions** across 5 categories, each encoded as a 5-bit `oper_type` field.

### Arithmetic Operations

| Opcode (Binary) | Mnemonic | Operation | Example |
|---|---|---|---|
| `5'b00000` | `movsgpr` | Move from/to SGPR | `movsgpr r1, SGPR` |
| `5'b00001` | `mov` | Load register or immediate | `mov r0, #5` |
| `5'b00010` | `add` | Addition | `add r2, r0, r1` |
| `5'b00011` | `sub` | Subtraction | `sub r3, r3, #1` |
| `5'b00100` | `mul` | Multiplication | `mul r4, r2, r3` |

### Logical Operations

| Opcode (Binary) | Mnemonic | Operation |
|---|---|---|
| `5'b00101` | `ror` | Bitwise OR |
| `5'b00110` | `rand` | Bitwise AND |
| `5'b00111` | `rxor` | Bitwise XOR |
| `5'b01000` | `rxnor` | Bitwise XNOR |
| `5'b01001` | `rnand` | Bitwise NAND |
| `5'b01010` | `rnor` | Bitwise NOR |
| `5'b01011` | `rnot` | Bitwise NOT |

### Memory / IO Operations

| Opcode (Binary) | Mnemonic | Operation |
|---|---|---|
| `5'b01101` | `storereg` | Store register to data memory |
| `5'b01110` | `storedin` | Store immediate to data memory |
| `5'b01111` | `senddout` | Send data to output port |
| `5'b10000` | `sendreg` | Send register to output port |

### Branch / Jump Operations

> The processor implements **9 jump variants** — 1 unconditional and 8 flag-based conditionals covering all combinations of the 4 ALU status flags.

| Opcode (Binary) | Mnemonic | Condition |
|---|---|---|
| `5'b10010` | `jump` | Unconditional jump |
| `5'b10011` | `jcarry` | Jump if carry flag set |
| `5'b10100` | `jnocarry` | Jump if carry flag clear |
| `5'b10101` | `jsign` | Jump if sign flag set (negative result) |
| `5'b10110` | `jnosign` | Jump if sign flag clear |
| `5'b10111` | `jzero` | Jump if zero flag set |
| `5'b11000` | `jnozero` | Jump if zero flag clear |
| `5'b11001` | `joverflow` | Jump if overflow flag set |
| `5'b11010` | `jnooverflow` | Jump if overflow flag clear |

### Control

| Opcode (Binary) | Mnemonic | Operation |
|---|---|---|
| `5'b11011` | `halt` | Stop execution |

---

## 🔀 Addressing Modes

The processor supports two addressing modes, selected by `IR[16]`:

```
MODE 0 — Register-Register  (imm_mode = 0)
─────────────────────────────────────────
  oper_type  rdst   rsrc1  0   rsrc2
  [31:27]  [26:22] [21:17] │  [15:11]
                            │
                    Both operands come from the register file.
                    Example: ADD r2, r0, r1

MODE 1 — Immediate  (imm_mode = 1)
─────────────────────────────────────────
  oper_type  rdst   rsrc1  1        isrc [15:0]
  [31:27]  [26:22] [21:17] │   16-bit constant in the instruction
                            │
                    Second operand is embedded in the instruction word.
                    Example: MOV r0, #5  →  isrc = 0x0005
```

---

## 🔬 Test Program — Multiply via Repeated Addition

To verify the processor, a **9-instruction multiply program** was assembled and loaded into `inst_data.mem`. It computes `5 × 6` using a loop — the simplest non-trivial program that exercises `mov`, `add`, `sub`, `jump`, and `halt` together.

### Algorithm

```
Compute:  r0 × r1  →  r4
Strategy: r2 += r1  (repeat r0 times, using r3 as counter)
```

### Instruction Listing

```
Addr  Hex         Binary                             Assembly              Notes
────  ──────────  ─────────────────────────────────  ────────────────────  ────────────────────────
[0]   0x08010005  00001000000000010000000000000101   MOV  r0, #5          Load multiplicand
[1]   0x08410006  00001000010000010000000000000110   MOV  r1, #6          Load multiplier
[2]   0x08810000  00001000100000010000000000000000   MOV  r2, #0          Init accumulator
[3]   0x08C10005  00001000110000010000000000000101   MOV  r3, #5          Loop counter = r0
[4]   0x10840800  00010000100001000000100000000000   ADD  r2, r2, r1      r2 += r1  ← LOOP START
[5]   0x18C70001  00011000110001110000000000000001   SUB  r3, r3, #1      r3--
[6]   0x90010004  10010000000000010000000000000100   JUMP #4              Branch back to [4]
[7]   0x09040000  00001001000001000000000000000000   MOV  r4, r2          r4 = result
[8]   0xD8000000  11011000000000000000000000000000   HALT                 Stop
```

### Execution Trace

```
Cycle  PC   Instruction     r0  r1  r2   r3  r4
─────  ──   ─────────────   ──  ──  ──   ──  ──
  1    0    MOV r0, #5       5   X   X    X   X
  2    1    MOV r1, #6       5   6   X    X   X
  3    2    MOV r2, #0       5   6   0    X   X
  4    3    MOV r3, #5       5   6   0    5   X
  5    4    ADD r2, r2, r1   5   6   6    5   X   ← iteration 1
  6    5    SUB r3, r3, #1   5   6   6    4   X
  7    6    JUMP #4          —   —   —    —   —
  8    4    ADD r2, r2, r1   5   6   12   4   X   ← iteration 2
  9    5    SUB r3, r3, #1   5   6   12   3   X
 10    6    JUMP #4
 11    4    ADD r2, r2, r1   5   6   18   3   X   ← iteration 3
 12    5    SUB r3, r3, #1   5   6   18   2   X
 13    6    JUMP #4
 14    4    ADD r2, r2, r1   5   6   24   2   X   ← iteration 4
 15    5    SUB r3, r3, #1   5   6   24   1   X
 16    6    JUMP #4
 17    4    ADD r2, r2, r1   5   6   30   1   X   ← iteration 5
 18    5    SUB r3, r3, #1   5   6   30   0   X
 19    6    JUMP #4
 20    4    ADD r2, r2, r1   5   6   30   0   X   ← r3 = 0, halt cond
 21    7    MOV r4, r2       5   6   30   0  30
 22    8    HALT
```

### inst_data.mem

```
00001000000000010000000000000101
00001000010000010000000000000110
00001000100000010000000000000000
00001000110000010000000000000101
00010000100001000000100000000000
00011000110001110000000000000001
10010000000000010000000000000100
00001001000001000000000000000000
11011000000000000000000000000000
```

---

## 📊 Simulation Results

### Register File Output (Waveform)

```
Signal       Final Value   Description
──────────   ───────────   ─────────────────────────────
rf[0][15:0]       5        Multiplicand — loaded by MOV r0, #5
rf[1][15:0]       6        Multiplier   — loaded by MOV r1, #6
rf[2][15:0]      30        Accumulator  — result of 5 additions of 6
rf[3][15:0]       0        Loop counter — decremented to 0
rf[4][15:0]      30        ✅ Final result — 5 × 6 = 30
SGPR[15:0]        X        Not written in this program
```

### Instruction Memory Read-Back (Data Bus Waveform)

The 32-bit instruction words read back from memory match the assembled encoding exactly:

```
Address   Decimal        Hex          Instruction
───────   ───────────    ──────────   ────────────
  [0]     134283269      0x08010005   MOV r0, #5
  [1]     138477574      0x08410006   MOV r1, #6
  [2]     142671872      0x08810000   MOV r2, #0
  [3]     146866181      0x08C10005   MOV r3, #5
  [4]     277088256      0x10840800   ADD r2, r2, r1
  [5]     415694849      0x18C70001   SUB r3, r3, #1
  [6]     2415984644     0x90010004   JUMP #4
  [7]     151257088      0x09040000   MOV r4, r2
  [8]     3623878656     0xD8000000   HALT
```

### Verification

```
Expected:  5 × 6 = 30
Got:       rf[4] = 30  ✅  PASS
```

---

## 📁 File Structure

```
risc-processor/
│
├── top.v                  ← Top-level processor module (fetch → decode → execute → writeback)
├── inst_data.mem          ← Instruction memory initialisation file (binary, one word per line)
│
├── README.md              ← This file
│
└── sim/
    ├── waveform_regs.png  ← Register file waveform (rf[0]–rf[7], SGPR)
    └── waveform_imem.png  ← Instruction memory read-back waveform
```

---

## ▶️ How to Run

### Prerequisites

- Any Verilog simulator: **ModelSim**, **Icarus Verilog (iverilog)**, **Vivado Simulator**, or **EDA Playground**

### With Icarus Verilog (free, open source)

```bash
# Compile
iverilog -o risc_proc top.v

# Simulate
vvp risc_proc

# View waveform (requires GTKWave)
gtkwave dump.vcd
```

### With ModelSim

```tcl
vlog top.v
vsim top
run -all
```

### With EDA Playground

1. Paste `top.v` into the **Design** pane
2. Set `inst_data.mem` as a simulation file
3. Run — waveforms appear in EPWave

---

## 🧩 Key Verilog Concepts Used

| Concept | Usage in this project |
|---|---|
| `` `define `` macros | Named IR bit-field slices for readable decode logic |
| `$readmemb` | Load binary instruction memory from `.mem` file at simulation start |
| `always @(posedge clk)` | Synchronous register file writeback and PC update |
| `case (oper_type)` | Instruction decode and ALU operation selection |
| Status flags | 4-bit flag register updated after every ALU operation |
| `initial` block | Register file and memory initialisation |

---

## 📌 What I Learned

- Getting **instruction fetch → decode → execute → writeback timing** right in RTL is fundamentally different from software — one misaligned clock edge produces garbage register values
- The `imm_mode` single-bit design keeps the instruction word compact without needing separate instruction format decoders
- **Simulation-first design is non-negotiable** — the waveform is your ground truth
- Flag-based conditional branching is what separates a toy processor from one that can run real algorithms

---

## 🔮 Planned Extensions

- [ ] 5-stage pipeline (IF → ID → EX → MEM → WB)
- [ ] Data hazard detection and forwarding paths
- [ ] Branch prediction (1-bit history)
- [ ] Memory-mapped IO controller
- [ ] Assembler script (Python) to convert assembly text → `.mem` file

---

## 👤 Author

**Anand** — VLSI aspirant, electronics and embedded systems enthusiast.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?style=flat&logo=linkedin)](https://linkedin.com/in/YOUR-PROFILE)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-black?style=flat&logo=github)](https://github.com/YOUR-USERNAME)

---

<div align="center">

*Designed from first principles. Verified in simulation. Built to learn.*

</div>
