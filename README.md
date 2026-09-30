# 8bit-alu-fifo-verilog
8-bit ALU integrated with a FIFO buffer in Verilog HDL, verified with ModelSim and synthesized in Quartus II.
<div align="center">

# 8-bit ALU + FIFO in Verilog

**A modular RTL design that connects a combinational ALU to a clocked FIFO buffer, verified through simulation.**

![Language](https://img.shields.io/badge/Language-Verilog%20HDL-blue)
![Synthesis](https://img.shields.io/badge/Synthesis-Quartus%20II-orange)
![Simulation](https://img.shields.io/badge/Simulation-ModelSim--Altera-green)
![Status](https://img.shields.io/badge/Status-Functionally%20Verified-brightgreen)

</div>

---

## Table of Contents

1. [Overview](#1-overview)
2. [Key Features](#2-key-features)
3. [System Architecture](#3-system-architecture)
4. [Repository Structure](#4-repository-structure)
5. [Module Specifications](#5-module-specifications)
6. [Verification](#6-verification)
7. [Getting Started](#7-getting-started)
8. [Known Limitations](#8-known-limitations)
9. [Future Work](#9-future-work)
10. [Skills Demonstrated](#10-skills-demonstrated)
11. [Author](#11-author)

---

## 1. Overview

This project implements an **8-bit Arithmetic Logic Unit (ALU)** and an **8-bit First-In First-Out (FIFO) buffer** in Verilog HDL and integrates them through a top-level module. The ALU computes a result from two 8-bit operands and a 3-bit opcode; that result is written into the FIFO, which returns stored values in the order they arrived.

The design pairs **combinational logic** (ALU) with **sequential, clock-controlled logic** (FIFO), and demonstrates module hierarchy, structural interconnection, clock/reset handling, and simulation-based functional verification.

| | |
|---|---|
| **Design language** | Verilog HDL |
| **Data width** | 8 bits |
| **Synthesis tool** | Quartus II 32-bit |
| **Simulation tool** | ModelSim-Altera Starter Edition |
| **Synthesis top module** | `alu_fifo_top` |
| **Simulation top module** | `alu_fifo_tb` |

### Motivation

Most beginner Verilog projects build a single block in isolation. This project combines two different kinds of hardware: **combinational logic** (the ALU, whose output changes as soon as its inputs change) and **sequential logic** (the FIFO, which stores data on clock edges). Connecting them shows how separate modules are integrated into a working system and how their timing behaviors differ.

### Design Approach

- The **ALU** takes two 8-bit operands and a 3-bit opcode and selects the operation with a `case` statement.
- The **FIFO** stores ALU results on the clock using `write_en` and `read_en`, and reports its state through the `full` and `empty` flags.
- A **top-level module** connects the two blocks with named ports, so the ALU result flows directly into the FIFO.
- A **clocked testbench** applies test vectors and checks the outputs in ModelSim.

### Results Summary

- ADD, SUB, AND, and OR produce the expected results (`0F`, `0C`, `88`, `EE`).
- The FIFO returns values in the same order they were written.
- The `empty` flag is high at start, low after writes, and high again once all values are read.

### What I Learned

- The difference between combinational and clocked behavior, and why it matters when connecting modules.
- How to split a design into synthesizable modules plus a simulation-only testbench.
- How to read waveforms and debug undefined modules, wrong top-level selection, and uninitialized signals.
- Passing a few directed tests is a starting point, not proof that a design is complete.

---

## 2. Key Features

- 8-bit ALU with a 3-bit opcode selecting the operation
- Status outputs: `carry` and `zero`
- Clock- and reset-controlled FIFO with `write_en` / `read_en` handshake
- `full` and `empty` status flags
- Clean top-level integration using named port connections
- Clocked testbench with directed test vectors and waveform inspection

## 3. System Architecture

### Data Flow

```
A[7:0]  --------\
                 >---- ALU ---- result_wire ---- FIFO ----> data_out[7:0]
B[7:0]  --------/        |
                         +----> alu_result[7:0]

opcode[2:0]                  ---> selects ALU operation
clk / reset / write_en / read_en ---> FIFO control
full / empty                 ---> FIFO status
```

### Module Hierarchy

```
alu_fifo_tb          (testbench, simulation only)
    |
    v
alu_fifo_top         (top-level integration)
   /      \
  v        v
 alu      fifo
```

## 4. Repository Structure

```
alu-fifo-verilog/
├── rtl/
│   ├── alu.v               # ALU module
│   ├── fifo.v              # FIFO module
│   └── alu_fifo_top.v      # Top-level integration
├── tb/
│   └── alu_fifo_tb.v       # Testbench
├── docs/
│   └── waveform.png        # Simulation waveform screenshot
└── README.md
```

| File | Module | Type | Purpose |
|------|--------|------|---------|
| `alu.v` | `alu` | Synthesizable | Arithmetic and logical operations |
| `fifo.v` | `fifo` | Synthesizable | Ordered storage and retrieval of results |
| `alu_fifo_top.v` | `alu_fifo_top` | Synthesizable | Connects the ALU to the FIFO |
| `alu_fifo_tb.v` | `alu_fifo_tb` | Simulation only | Stimulus generation and output checking |

## 5. Module Specifications

### 5.1 ALU

**Ports**

| Signal | Width | Direction | Description |
|--------|:-----:|:---------:|-------------|
| `A` | 8 | Input | First operand |
| `B` | 8 | Input | Second operand |
| `opcode` | 3 | Input | Operation select |
| `result` | 8 | Output | Operation result |
| `carry` | 1 | Output | Carry/status flag |
| `zero` | 1 | Output | Asserted when `result` is zero |

**Operation table**

| Opcode | Operation | Expression |
|:------:|-----------|------------|
| `000` | ADD | `A + B` |
| `001` | SUB | `A - B` |
| `010` | AND | `A & B` |
| `011` | OR | `A \| B` |
| others | Default | `8'b0` |

**Implementation**

```verilog
case (opcode)
    3'b000: result = A + B;
    3'b001: result = A - B;
    3'b010: result = A & B;
    3'b011: result = A | B;
    default: result = 8'b0;
endcase
```

The ALU is purely combinational: `result` updates whenever `A`, `B`, or `opcode` changes.

### 5.2 FIFO

The FIFO is a queue: the first value written is the first value read. It decouples the ALU (which produces results) from any consumer that reads them later.

**Ports**

| Signal | Direction | Description |
|--------|:---------:|-------------|
| `clk` | Input | Clock; synchronizes state changes |
| `reset` | Input | Initializes the FIFO |
| `data_in` | Input | Data to be stored |
| `write_en` | Input | Write request |
| `read_en` | Input | Read request |
| `data_out` | Output | Data returned on read |
| `full` | Output | FIFO has reached capacity |
| `empty` | Output | FIFO holds no valid data |

### 5.3 Top-Level Integration

`alu_fifo_top` instantiates both modules. The internal wire `result_wire` carries the ALU result into the FIFO and also drives the `alu_result` output.

```verilog
wire [7:0] result_wire;
assign alu_result = result_wire;

alu alu (
    .A(A), .B(B), .opcode(opcode),
    .result(result_wire), .carry(carry), .zero(zero)
);

fifo fifo (
    .clk(clk), .reset(reset),
    .data_in(result_wire),
    .write_en(write_en), .read_en(read_en),
    .data_out(data_out), .full(full), .empty(empty)
);
```

Named port connections (for example `.result(result_wire)`) keep wiring explicit and reduce connection errors.

## 6. Verification

The testbench instantiates `alu_fifo_top` as the design under test (DUT), drives a clock, applies directed test vectors, and observes outputs in ModelSim (waveform and transcript).

### 6.1 ALU Test Results

| Test | A | B | Opcode | Expected | Result |
|------|:--:|:--:|:------:|:--------:|:------:|
| ADD | `0A` | `05` | `000` | `0F` | Pass |
| SUB | `14` | `08` | `001` | `0C` | Pass |
| AND | `AA` | `CC` | `010` | `88` | Pass |
| OR | `AA` | `CC` | `011` | `EE` | Pass |

### 6.2 FIFO Ordering Test

```
WRITE : 0F -> 0C -> 88 -> EE
READ  : 0F -> 0C -> 88 -> EE
```

Values were read back in the same order they were written, confirming correct first-in first-out behavior.

### 6.3 Flag Behavior

| Condition | `empty` | `full` |
|-----------|:-------:|:------:|
| After reset (start of test) | 1 | 0 |
| After valid writes | 0 | 0 |
| After all four values are read | 1 | 0 |

`full` remained low because the FIFO was never filled to capacity during this test.

### 6.4 Waveform

<img width="1091" height="412" alt="Screenshot 2026-09-30 125755" src="https://github.com/user-attachments/assets/1cbe9a4d-97f0-4a60-803f-349443c6c7b6" />
<img width="895" height="400" alt="Screenshot 2026-09-30 125904" src="https://github.com/user-attachments/assets/1cf68f71-5194-4ecf-816b-dd97be49eb3f" />

*ModelSim waveform showing ALU results being written to and read from the FIFO.*

## 7. Getting Started

### Prerequisites

- Quartus II (32-bit) for synthesis
- ModelSim-Altera Starter Edition for simulation

### Run the Simulation (ModelSim)

1. Create a new ModelSim project.
2. Add `alu.v`, `fifo.v`, `alu_fifo_top.v`, and `alu_fifo_tb.v`.
3. Compile all files.
4. Simulate `alu_fifo_tb`.
5. Add signals to the waveform window and run the simulation.
6. Verify results in the waveform and transcript.

### Run Synthesis (Quartus II)

1. Create a new Quartus project.
2. Add `alu.v`, `fifo.v`, and `alu_fifo_top.v` (exclude the testbench).
3. Set `alu_fifo_top` as the top-level entity.
4. Run **Analysis and Synthesis**.



## 8. Known Limitations

- Only four ALU operations are implemented and tested (ADD, SUB, AND, OR); remaining opcodes return zero.
- Verification uses directed tests only; there is no randomized or self-checking test yet.
- The full condition and empty-read (underflow) behavior are not exercised by the current testbench.
- Signed overflow, borrow, and negative flags are not implemented.
- The design has been verified in simulation only, not on hardware.

## 9. Future Work

- Add operations: XOR, NOT, shifts, compare
- Add borrow, overflow, and negative flags
- Add overflow/underflow protection and almost-full / almost-empty flags to the FIFO
- Build a self-checking testbench with randomized inputs and a scoreboard
- Add tests for the full FIFO and read-when-empty cases
- Implement and demonstrate on an FPGA board
- Extend verification to SystemVerilog/UVM with assertions and functional coverage

## 10. Skills Demonstrated

- RTL design in Verilog HDL
- Combinational vs. sequential logic design
- Hierarchical design and module instantiation
- Testbench development and waveform-based debugging
- Quartus synthesis and ModelSim simulation flow

## 11. Author

**Jaimin Doshi**
B.Tech, Electronics and Communication Engineering

- GitHub: [click-here](https://github.com/JAIMIN2711)
- LinkedIn: [click-here](https://www.linkedin.com/in/jaimin-doshi)
- Email: doshijaimin27@gmail.com
