# SKY130 RTL Design and Synthesis Workshop – Day 2

Timing libraries, hierarchical vs flat synthesis, flip-flop coding styles and RTL optimization using **Yosys**, **Icarus Verilog**, **GTKWave** and the **SkyWater SKY130** standard cell library.

## Table of Contents
1. [Introduction to the `.lib` timing library](#1-introduction-to-the-lib-timing-library)
2. [Hierarchical vs Flat synthesis](#2-hierarchical-vs-flat-synthesis)
3. [Flip-flops: why and how to code them](#3-flip-flops-why-and-how-to-code-them)
4. [RTL optimization](#4-rtl-optimization)

> Screenshots are stored in the `docs/images/` folder and named by the notes numbering (e.g. `14_1.png`).

---

## 1. Introduction to the `.lib` timing library

Open the library:

```bash
gvim ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```

![11.1 – .lib file](docs/images/11_1.png)

**Library name:** `sky130_fd_sc_hd__tt_025C_1v80`

| Part | Meaning |
|------|---------|
| `tt` | Typical process corner |
| `025C` | Temperature = 25 °C |
| `1v80` | Voltage = 1.8 V |

### PVT (Process, Voltage, Temperature)
- **Process** – variation due to fabrication.
- **Voltage** – supply variation changes the behaviour/function of the cell.
- **Temperature** – determines how the silicon behaves.
- PVT is important for a design to work reliably under all conditions.

### What the `.lib` file tells us
- Technology (`cmos`)
- Units of time, voltage, power and current
- Operating conditions (voltage, process, temperature)
- All cells with their different flavours, e.g. `a2111o_1` implements `Y = (A1·A2) + B1 + C1 + D1` → 5 inputs → 2⁵ = 32 possible input combinations, each needing power and delay data.

### Fast cells vs slow cells
- Smaller cell → more delay, less area, less power.
- Faster cell → more area, consumes more power.

**Note on CMOS:** a NAND gate has stacked NMOS (good). A NOR gate has stacked PMOS, and PMOS has poor mobility, so the cell needs to be sized up to get good logic. An OR gate = NOR followed by an inverter.

---

## 2. Hierarchical vs Flat synthesis

### 2.1 Design: `multiple_modules.v`

Two sub-modules (AND, OR) used inside a top module with 3 inputs (a, b, c) and one output y: `y = (a & b) | c`.

```bash
gvim multiple_modules.v
```

![14.1 – multiple_modules.v](docs/images/14_1.png)

### 2.2 Hierarchical synthesis

```bash
yosys
read_liberty -lib ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog multiple_modules.v
synth -top multiple_modules
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
show multiple_modules
```

![14.2 – show multiple_modules (hierarchy preserved: u1, net1, u2)](docs/images/14_2.png)

Write the hierarchical netlist:

```bash
write_verilog -noattr multiple_modules_hier.v
!gvim multiple_modules_hier.v
```

![14.3 – hierarchical netlist](docs/images/14_3.png)

### 2.3 Flat netlist

```bash
flatten
write_verilog -noattr multiple_modules_flat.v
!gvim multiple_modules_flat.v
```

![15.1 – flat netlist](docs/images/15_1.png)

In a flat netlist the AND and OR gates appear **directly under the top module**, whereas in the hierarchical netlist there is a separate sub-module for the AND and the OR gate.

```bash
show
```

![15.2 – show after flatten](docs/images/15_2.png)

### 2.4 Sub-module level synthesis

```bash
yosys
read_liberty -lib ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog multiple_modules.v
synth -top sub_module1
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

![15.3 – sub_module1 only (AND gate)](docs/images/15_3.png)

Only the AND gate is shown because `synth -top` controls which module is synthesized.

**Why sub-module level synthesis?**
1. **Multiple instances of the same module:** synthesize the module once and replicate/stitch it, instead of synthesizing it 6 times.
2. **Massive designs:** give it in portions and then stitch it together; `synth -top` controls which module goes to synthesis.

---

## 3. Flip-flops: why and how to code them

### 3.1 Why flops?
A combinational circuit output changes after the propagational delay. With multiple continuous combinational circuits, input changes can produce **glitches** at the output (e.g. `y = (a & b) | c`, when `a`, `b`, `c` change at slightly different times). To avoid this, a flip-flop is used to store the value: the output of a flop only changes on the clock edge, so the glitchy output stays stable until the next edge.

The flop needs to be initialised with a **reset / set**, which can be **asynchronous** or **synchronous**.

### 3.2 Coding styles

**Asynchronous reset** – `q` goes to 0 irrespective of the clock.

```verilog
module dff_asyncres (input clk, input async_reset, input d, output reg q);
  always @(posedge clk, posedge async_reset)
  begin
    if (async_reset)
      q <= 1'b0;
    else
      q <= d;
  end
endmodule
```

**Asynchronous set**

```verilog
module dff_async_set (input clk, input async_set, input d, output reg q);
  always @(posedge clk, posedge async_set)
  begin
    if (async_set)
      q <= 1'b1;
    else
      q <= d;
  end
endmodule
```

**Synchronous reset** – the reset is only sampled on the clock edge.

```verilog
module dff_syncres (input clk, input async_reset, input sync_reset, input d, output reg q);
  always @(posedge clk)
  begin
    if (sync_reset)
      q <= 1'b0;
    else
      q <= d;
  end
endmodule
```

### 3.3 Simulation

```bash
iverilog dff_asyncres.v tb_dff_asyncres.v
./a.out
gtkwave tb_dff_asyncres.vcd
```

Asynchronous reset: when reset goes high, `q` also goes low **without considering the clock**.

![18.1 – tb_dff_asyncres waveform](docs/images/18_1.png)

Asynchronous set: when `async_set` is high, `q` goes high regardless of the clock. Upon the clock edge, `d` is the same as `q` when async set is low.

```bash
iverilog dff_async_set.v tb_dff_async_set.v
./a.out
gtkwave tb_dff_async_set.vcd
```

![18.2 – tb_dff_async_set waveform](docs/images/18_2.png)

Synchronous reset: `q` only changes on the clock edge. When `sync_reset = 1`, `q` is cleared on the next clock edge.

```bash
iverilog dff_syncres.v tb_dff_syncres.v
./a.out
gtkwave tb_dff_syncres.vcd
```

![18.3 – tb_dff_syncres waveform](docs/images/18_3.png)

### 3.4 Synthesis of the flops

```bash
yosys
read_liberty -lib ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog dff_asyncres.v
synth -top dff_asyncres
dfflibmap -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

**Asynchronous reset** → mapped to `sky130_fd_sc_hd__dfrtp_1` (active-low `RESET_B`), with an inverter (`clkinv_1`) on `async_reset`.

![19.1 – dff_asyncres synthesis](docs/images/19_1.png)

**Asynchronous set** (`dff_async_set.v`) → mapped to `sky130_fd_sc_hd__dfstp_2` (`SET_B`) with an inverter.

![19.2 – dff_async_set synthesis](docs/images/19_2.png)

**Synchronous reset** (`dff_syncres.v`) → plain D flop `sky130_fd_sc_hd__dfxtp_1`. There is **no set/reset pin**; the reset is implemented in logic in front of D (`nor2b_1`):

`D = ~sync_reset & d`

![19.3 – dff_syncres synthesis](docs/images/19_3.png)

---

## 4. RTL optimization

### 4.1 Multiply by 2 – `mul2`

```verilog
module mul2 (input [2:0] a, output [3:0] y);
  assign y = a * 2;
endmodule
```

```bash
gvim mult_*.v -o
```

![20.1 – mult_2.v and mult_8.v](docs/images/20_1.png)

```bash
yosys
read_liberty -lib ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog mult_2.v
synth -top mul2
abc -liberty ../lib/sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

Result: **no cells, no ABC mapping**. Multiplying by 2 is just a left shift, so the output is the input with a `0` appended (`y = {a, 1'b0}`) – it is only wiring.

![20.2 – show mul2](docs/images/20_2.png)

```bash
write_verilog -noattr mul2_net.v
!gvim mul2_net.v
```

![21.1 – mul2 netlist: `assign y = { a, 1'h0 };`](docs/images/21_1.png)

### 4.2 Multiply by 9 – `mult8`

```verilog
module mult8 (input [2:0] a, output [5:0] y);
  assign y = a * 9;
endmodule
```

Since `a × 9 = a × (8 + 1) = (a × 8) + (a × 1)`, this is `a` shifted left by 3 bits concatenated with `a` → `y = {a, a}`. Again no logic cells are needed.

```bash
read_verilog mult_8.v
synth -top mult8
write_verilog -noattr mult8_net.v
!gvim mult8_net.v
show
```

![21.2 – mult8 netlist and show (`assign y = { a, a };`)](docs/images/21_2.png)

---

## Tools used
- [Yosys](https://github.com/YosysHQ/yosys) 0.52
- Icarus Verilog
- GTKWave
- SkyWater SKY130 `sky130_fd_sc_hd__tt_025C_1v80.lib`
