# Runtime-Programmable Multi-Protocol Emulator

A reprogrammable communication engine supporting **UART, SPI, I2C, JTAG, PS/2 and SWD on one shared hardware core**.

Instead of building a separate controller for every protocol, we built reusable timing, data-shifting and pin-control hardware. Protocol behavior is stored in writable memory, so loading a different program changes how the same hardware communicates.

The design was developed on a **DE10-Lite FPGA** and later adapted for ASIC implementation through **Tiny Tapeout**, including gate-level verification and GDS generation.

**Verilog · 50 MHz · DE10-Lite · Tiny Tapeout**

---

## The Main Idea

UART, SPI, I2C and the other supported protocols work differently, but they repeatedly need many of the same basic operations:

- send or read a bit
- wait for a set amount of time
- generate clock edges
- control external pins
- store received data
- check how another device responded
- choose what to do next

Instead of rebuilding those functions for each protocol, we implemented them once.

```text
                 Program Memory
                       |
                       v
              +-----------------+
              |  Protocol Core  |
              +-----------------+
                 |     |     |
                 v     v     v
              Timer  Shifter  I/O
                 \     |     /
                  \    |    /
                   Physical Pins
```

The program in memory determines how those blocks are used.

A UART program sends a start bit, data and a stop bit. An SPI program uses the same hardware to control a clock and transfer data. An I2C program can send an address, wait for the connected device to respond and choose what to do next.

The hardware stays largely the same. **The loaded program changes the protocol.**

---

## Why Make It Reprogrammable?

The same hardware can be reused across different devices without building another controller every time.

If one board needs UART and another needs SPI, we load another program. If an I2C device is not responding, we can change its address, timing, transmitted data or expected response in the program and test again.

On the FPGA, this avoids repeating:

**edit RTL → compile → check timing → generate bitstream → reprogram FPGA**

when the hardware already supports the operations we need.

It also helps with debugging. We can change the communication sequence while keeping the underlying hardware fixed, which makes it easier to separate problems caused by the protocol, timing, wiring or connected device.

The ASIC version keeps the same feature through a runtime loader, so the protocol behavior can still be changed after fabrication.

This approach could be useful in electronics test equipment, robots using parts from different vendors, automotive electronics, or products that need to communicate with different hardware over time.

---

## How the Core Works

The program is built from simple communication operations:

1. **Drive a pin** HIGH or LOW
2. **Read a pin**
3. **Wait** for a set amount of time
4. **Wait for a signal change**
5. **Load data**
6. **Send several bits**
7. **Read several bits**
8. **Send and receive at the same time**
9. **Move to another part of the program**
10. **Choose what to do based on a received value**
11. **Set timing, pins and bit order**
12. **Finish the transaction**

A UART program can be written as:

```text
set timing
load data
send start bit
send 8 data bits
send stop bit
finish
```

SPI rearranges many of the same operations:

```text
set clock behavior
load data
activate device
send 8 bits while generating clock
release device
finish
```

The final core uses a **64 × 16-bit writable program memory**, four small data registers, a timer, serial shifter and programmable I/O.

---

# Hard-Coded vs Programmable

This became the main architecture comparison in the project.

The straightforward way to support six protocols is to build six separate controllers.

That works, but every controller needs its own hardware for jobs that often overlap:

- timing
- data shifting
- registers
- pin control
- decision logic

UART may have its own data-shifting hardware while SPI has another and I2C has another. The protocols are different, but all three still need to move serial data.

Our design shares those resources.

Instead of adding another full controller when a protocol is added, much of the new behavior is stored as another program.

### FPGA Results

We synthesized conventional hard-coded versions and compared them against the programmable design.

| Protocols | Hard-Coded | Programmable |
|---:|---:|---:|
| 3 | **347 LEs** | **557 LEs** |
| 4 | **475 LEs** | **557 LEs** |
| 5 | **572 LEs** | **557 LEs** |
| 6 | **762 LEs** | **557 LEs** |

The programmable approach is **not automatically smaller**.

It carries extra hardware for program storage and execution, so a fixed design wins when only a few protocols are needed.

At five protocols, the two designs were roughly equal.

At six:

**762 LEs → 557 LEs**

That saved **205 logic elements, or about 27%**.

### Why the Hard-Coded Design Grows Faster

Adding another fixed controller usually adds more:

- timing hardware
- data-moving hardware
- registers
- control logic
- logic for selecting which controller is active

The programmable version keeps reusing hardware that is already there.

So instead of:

**another protocol → another controller**

the design moves closer to:

**another protocol → another program**

### Why Saving Logic Matters

An FPGA only has a fixed amount of hardware available.

Using **205 fewer logic elements** leaves more space for whatever else the FPGA needs to do, such as sensor processing, display control, memory interfaces or other hardware blocks.

A larger design also gives the FPGA tools more hardware and wiring to place and connect.

We did not separately measure routing congestion or power, so we do not claim improvements for either. The measured result is the logic reduction itself.

The programmable core also still met timing with **+9.537 ns setup slack at 50 MHz**, so sharing the hardware did not prevent the design from reaching its target clock.

### The Tradeoff

For a product that only needs two or three fixed protocols, dedicated controllers may make more sense.

For a system that needs many communication methods or needs to change behavior during debugging, the shared architecture becomes more useful.

Our crossover happened at roughly **five protocols**.

At six protocols we had:

- **6 communication standards**
- **557 LEs**
- **205 fewer LEs than the hard-coded version**
- **~27% logic reduction**
- **runtime reprogramming**
- **positive timing slack at 50 MHz**

---

## Key Results

| Metric | Result |
|---|---:|
| Supported protocols | **6** |
| Programmable design | **557 LEs** |
| 6-protocol hard-coded design | **762 LEs** |
| Logic saved | **205 LEs / ~27%** |
| Registers | **165** |
| Program memory | **1,024 bits** |
| FPGA clock | **50 MHz** |
| Setup slack | **+9.537 ns** |
| Gate-level tests | **6 / 6 passed** |
| Final routing DRC errors | **0** |
| LVS / Antenna | **Passed** |
| ASIC output | **GDS generated** |

---

## Protocols Tested

| Protocol | What We Tested |
|---|---|
| **UART** | Start bit, 8-bit transfer and stop bit |
| **SPI** | Mode 0 transfer, clocking and device select |
| **I2C** | Start, device address, response check and stop |
| **PS/2** | Start, data, parity and stop frame |
| **SWD** | Request, device response and data transfer |
| **JTAG** | TAP state movement and data-register scan |

---

## FPGA Verification & Hardware Demo

We first tested the RTL using **Icarus Verilog** and inspected the generated signals in **GTKWave**.

Tests checked transmitted data, clock edges, bit order, device responses and shared-pin behavior rather than only checking whether the program finished.

The same RTL was then compiled in **Quartus Prime** and loaded onto the **DE10-Lite**.

Final FPGA core:

- **557 logic elements**
- **165 registers**
- **1,024 memory bits**
- **+9.537 ns setup slack at 50 MHz**

The DE10-Lite seven-segment displays showed the selected protocol, execution status and sent/received data.

This helped during hardware debugging. A transaction could run correctly inside the FPGA while the connected device still failed to answer because of wiring, an incorrect address or a missing connection.

---

## FPGA to ASIC

After the FPGA version worked, we adapted the same core for Tiny Tapeout.

The main changes were:

- replacing FPGA-specific bidirectional pin handling with separate input, output and output-enable signals
- removing the Intel-specific memory setting
- adapting the reset
- replacing the wide FPGA programming interface with an **8-bit program loader**

The loader receives two bytes, combines them into one 16-bit instruction and writes it into program memory.

This kept the main feature of the project intact: **the ASIC remains reprogrammable after fabrication.**

---

## ASIC Verification & Physical Design

The ASIC version reused the same six protocol programs.

**Cocotb** loaded each program through the ASIC loader and checked the resulting communication signals.

**6 / 6 gate-level protocol tests passed.**

The design was then taken through physical implementation and produced:

- **0 final routing DRC errors**
- **LVS passed**
- **Antenna checks passed**
- **0 setup TNS**
- **0 hold TNS**
- **Final GDS generated**

The GDS is the placed-and-routed physical implementation of the same programmable core developed on the FPGA.

---

## What We Learned

**Shared hardware has a break-even point.** The programmable version was larger with only three or four protocols, roughly equal at five, and smaller at six.

**Finishing a program does not prove the protocol worked.** We had to check the actual data, timing and device responses.

**External hardware can look like an RTL problem.** Wrong wiring, missing pull-ups or a device that does not respond can make a working core appear broken.

**FPGA and ASIC constraints are different.** Pin count, memory, reset and bidirectional I/O all needed changes during the ASIC migration.

---

## Next Steps

- Build a simple tool that converts readable protocol steps into program memory
- Build a Python interface for changing timing, data and protocol programs
- Add automatic protocol detection by observing signals and testing likely device responses
- Perform post-silicon testing if fabricated hardware becomes available

---

## Tools

**Verilog · Quartus Prime · Icarus Verilog · GTKWave · Cocotb · Python · DE10-Lite · Tiny Tapeout · LibreLane · IHP CMOS5L**

---

## Team

**Ali Barrak · Aadit Mital**
