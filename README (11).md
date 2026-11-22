# Gate Level Simulation (GLS) & Synthesis–Simulation Mismatches

Hands-on experiments from the **SKY130 RTL Design and Synthesis Workshop** (Day 4). The goal is to understand what GLS is, why it is needed, and how coding mistakes in RTL (a missing sensitivity list, blocking vs. non-blocking assignments) can make the **RTL simulation** and the **synthesized netlist simulation** disagree.

**Tools used:** `iverilog`, `gtkwave`, `yosys`, `gvim`, SKY130 standard cell library (`sky130_fd_sc_hd`)

---

## Table of Contents

1. [Introduction to GLS](#1-introduction-to-gls)
2. [Synthesis–Simulation Mismatches](#2-synthesissimulation-mismatches)
3. [Lab 1 – Ternary operator MUX (good mux)](#3-lab-1--ternary-operator-mux-good-mux)
4. [Lab 2 – Bad MUX (missing sensitivity list)](#4-lab-2--bad-mux-missing-sensitivity-list)
5. [Lab 3 – Blocking assignment caveat](#5-lab-3--blocking-assignment-caveat)
6. [Conclusion](#6-conclusion)

---

## 1. Introduction to GLS

**What is GLS?**
- Running the testbench with the **netlist** as the Design Under Test (DUT).
- The netlist is logically the same as the RTL code, so the same testbench should work with it.

**Why GLS?**
- To verify the **logical correctness** of the design after synthesis.
- To ensure the **timing** of the design is met. For this, GLS must be run with **delay annotation**.

**GLS flow using iverilog**

```
Design (RTL) / Gate-level verilog model ─┐
                                         ├──► iverilog ──► VCD file ──► gtkwave
Testbench ───────────────────────────────┘
```

> **Note:** If the gate-level models are delay annotated, GLS can also be used for timing validation.

---

## 2. Synthesis–Simulation Mismatches

The simulator works on **activity**, while synthesis works on the **described logic**. The mismatches covered in this lab are:

| Cause | What goes wrong |
|---|---|
| Missing sensitivity list | Simulator only re-evaluates the output when a signal in the list changes. No activity means no evaluation, even though the synthesized hardware reacts to every input. |
| Blocking vs. non-blocking assignments | Statement order changes the result in simulation, which may not match what synthesis builds. |
| Non-standard Verilog coding | Behaviour is not guaranteed to be the same between simulation and synthesis. |

### Missing sensitivity list

```verilog
module mux (input i0, input i1, input sel, output reg y);
always @(sel)           // only sensitive to sel
begin
    if (sel)
        y = i1;
    else
        y = i0;
end
endmodule
```

Here `y` is only re-evaluated when `sel` changes. If `i0`/`i1` toggle while `sel` is constant, there is no activity on `sel`, so `y` does not update. Using `always @(*)` makes the block evaluate whenever **any** signal on the right-hand side changes.

### Blocking vs. non-blocking assignments (inside an `always` block)

- `=` → **Blocking**
  - Executes statements in the order they are written.
  - The first statement is evaluated before the second statement.
- `<=` → **Non-blocking**
  - When the `always` block is entered, all RHS values are evaluated first and then assigned to the LHS.
  - Parallel evaluation.

**Use non-blocking assignments when writing sequential circuits.**

### Caveat with blocking statements

```verilog
module code (input clk, input reset, input d, output reg q);
reg q0;
always @(posedge clk, posedge reset)
begin
    if (reset)
    begin
        q0 = 1'b0;
        q  = 1'b0;
    end
    else
    begin
        q  = q0;
        q0 = d;
    end
end
endmodule
```

With blocking assignments, the result depends on the order of the statements (`q = q0; q0 = d;` versus `q0 = d; q = q0;`), which changes how many flip-flops are inferred. Using non-blocking (`q0 <= d; q <= q0;`) removes this order dependence.

---

## 3. Lab 1 – Ternary operator MUX (good mux)

The ternary operator has the form `<cond> ? <true> : <false>`.

```verilog
assign y = sel ? i1 : i0;
```

**Commands**

```bash
# open the ternary mux and the good_mux side by side
gvim ternary_operator_mux.v -o good_mux.v

# RTL simulation
iverilog ternary_operator_mux.v tb_ternary_operator_mux.v
./a.out
gtkwave tb_ternary_operator_mux.vcd

# synthesis
yosys
read_liberty -lib <path-to>/sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog ternary_operator_mux.v
synth -top ternary_operator_mux
abc -liberty <path-to>/sky130_fd_sc_hd__tt_025C_1v80.lib
write_verilog -noattr ternary_operator_mux_net.v
show
```

**Source files (`ternary_operator_mux.v` and `good_mux.v`)** – `good_mux.v` uses `always @(*)` with `if (sel) y <= i1; else y <= i0;`

![Source files in gvim](docs/images/35_1.png)

**RTL simulation waveform** – `y` follows `i0` when `sel = 0` and `i1` when `sel = 1`.

![Ternary MUX RTL simulation](docs/images/35_2.png)

**Synthesized netlist** – ABC maps the design to a single `sky130_fd_sc_hd__mux2_1` cell (inputs `A0 = i0`, `A1 = i1`, `S = sel`, output `X = y`).

![Ternary MUX synthesized netlist](docs/images/35_3.png)

---

## 4. Lab 2 – Bad MUX (missing sensitivity list)

`bad_mux.v` is the MUX described with `always @(sel)` (see [section 2](#missing-sensitivity-list)).

### 4.1 RTL simulation

```bash
iverilog bad_mux.v tb_bad_mux.v
./a.out
gtkwave tb_bad_mux.vcd
```

In the waveform, **`y` only changes when `sel` changes**. It does not follow `i0` / `i1` while `sel` is constant.

![bad_mux RTL simulation](docs/images/36_1.png)

### 4.2 Synthesis and GLS

```bash
yosys
read_liberty -lib <path-to>/sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog bad_mux.v
synth -top bad_mux
abc -liberty <path-to>/sky130_fd_sc_hd__tt_025C_1v80.lib
write_verilog -noattr bad_mux_net.v
show

# Gate level simulation (netlist + SKY130 verilog models + testbench)
iverilog ../my_lib/verilog_model/primitives.v ../my_lib/verilog_model/sky130_fd_sc_hd.v bad_mux_net.v tb_bad_mux.v
./a.out
gtkwave tb_bad_mux.vcd
```

> If `iverilog` reports `No such file or directory` for the library files, check the relative path to the `verilog_model` folder. In this setup the correct folder was `../my_lib/`, not `../lib/`.

In the GLS waveform, **`y` now follows `i0` when `sel = 0` and `i1` when `sel = 1`**, i.e. the netlist behaves like a proper MUX. This does **not** match the RTL simulation above.

![bad_mux GLS](docs/images/36_2.png)

---

## 5. Lab 3 – Blocking assignment caveat

```verilog
module blocking_caveat (input a, input b, input c, output reg d);
reg x;
always @(*)
begin
    d = x & c;
    x = a | b;
end
endmodule
```

The intended logic is `d = (a | b) & c`, but because `d` is computed **before** `x` is updated, `d` uses the *previous* value of `x`.

### 5.1 RTL simulation

```bash
gvim blocking_caveat.v
iverilog blocking_caveat.v tb_blocking_caveat.v
./a.out
gtkwave tb_blocking_caveat.vcd
```

![blocking_caveat source in gvim](docs/images/37_1.png)

The waveform shows `d` depending on the old value of `x`, so it does not behave as a plain `(a | b) & c`.

![blocking_caveat RTL simulation](docs/images/37_2.png)

### 5.2 Synthesis

```bash
yosys
read_liberty -lib <path-to>/sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog blocking_caveat.v
synth -top blocking_caveat
abc -liberty <path-to>/sky130_fd_sc_hd__tt_025C_1v80.lib
write_verilog -noattr blocking_caveat_net.v
show
```

> Note: the option is `-noattr` (one word). Writing `-no attr` gives a *"Command syntax error: Unknown option"*.

Synthesis builds a single `sky130_fd_sc_hd__o21a_1` cell, which implements `X = (A1 | A2) & B1`. Here `A1 = b`, `A2 = a`, `B1 = c` and `X = d`, so the hardware is `d = (a | b) & c`.

![blocking_caveat synthesized netlist](docs/images/38_1.png)

### 5.3 Gate level simulation

```bash
iverilog ../my_lib/verilog_model/primitives.v ../my_lib/verilog_model/sky130_fd_sc_hd.v blocking_caveat_net.v tb_blocking_caveat.v
./a.out
gtkwave tb_blocking_caveat.vcd
```

The GLS waveform reflects the synthesized `(a | b) & c` logic and differs from the RTL simulation in 5.1.

![blocking_caveat GLS](docs/images/38_2.png)

---

## 6. Conclusion

- Synthesis builds hardware from the logic described in the code, while the RTL simulator behaves according to event/activity and statement order. Poor coding style can make these two disagree.
- In the **bad MUX** experiment, the RTL simulation (`always @(sel)`) only updated `y` on `sel` changes, whereas the gate-level simulation behaved like a normal 2:1 MUX.
- In the **blocking caveat** experiment, the RTL simulation used the stale value of `x`, whereas the synthesized `o21a` cell and its GLS implemented `(a | b) & c`.
- Running **GLS** with the same testbench is how these mismatches were caught, so it is a necessary step to check the logical correctness of a design after synthesis.
- To avoid such mismatches: use `always @(*)` for combinational logic (complete sensitivity list) and use non-blocking assignments (`<=`) for sequential logic.
