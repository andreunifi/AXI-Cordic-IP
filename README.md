# Approximate CORDIC IP Core (`Cordic_AXI`)

An AXI-Lite FPGA IP core implementing an **approximate CORDIC-based multiplier**, designed for low-power neuromorphic hardware. Based on [klab-aizu/IZH-ApproxCORD](https://github.com/klab-aizu/IZH-ApproxCORD).

---

## Overview

This IP core computes the product of two 8-bit integer operands using an approximate CORDIC algorithm. Replacing a conventional multiplier with a CORDIC-based approximation reduces area and power consumption, at the cost of a small, bounded numerical error — acceptable in noise-tolerant applications such as spiking neural networks.

The core exposes a simple AXI-Lite slave interface and signals completion via an interrupt line (IRQ), making it straightforward to integrate into Zynq-based SoC designs and drive from software (e.g. PYNQ).

---

## Interface

### AXI-Lite Register Map

| Offset | Name       | Access | Description                                      |
|--------|------------|--------|--------------------------------------------------|
| `0x00` | `slv_reg0` | R/W    | Operand X (8-bit)                                |
| `0x04` | `slv_reg1` | R/W    | Operand Z (8-bit)                                |
| `0x08` | `slv_reg2` | R/W    | Control — write `0` to reset, `3` to start       |
| `0x0C` | `slv_reg3` | R      | Result Y (16-bit signed, two's complement)       |

### Interrupt

The core raises its IRQ line when the computation is complete. The host must wait for the interrupt before reading `slv_reg3`.

---

## Operation Sequence

1. Write operand X to offset `0x00`.
2. Write operand Z to offset `0x04`.
3. Write `0` to offset `0x08` to reset the core.
4. Write `3` to offset `0x08` to start the computation.
5. Wait for the IRQ.
6. Read the 16-bit signed result from offset `0x0C`.

### PYNQ example

```python
cordic = overlay.Cordic_AXI_0

cordic.write(0x00, x & 0xFF)   # operand X
cordic.write(0x04, z & 0xFF)   # operand Z
cordic.write(0x08, 0)          # reset
cordic.write(0x08, 3)          # start

await cordic.IRQ.wait()        # wait for completion

raw = cordic.read(0x0C) & 0xFFFF
result = raw if raw < 0x8000 else raw - 0x10000  # sign-extend
```

---

## Result Interpretation

The output in `slv_reg3` is a 16-bit two's complement integer. When used for squaring (`X = Z = v`), the result should be rescaled to match the fixed-point format of the calling system. For example, with operands in Q0 (plain integers) and a host fixed-point scheme of Q6.9:

```python
v_8bit      = int(v_q69 / 512)            # downscale to integer
v_sq        = await cordic_compute(v_8bit, v_8bit)
v_sq_q69    = v_sq * 512                  # rescale result
```

---

## Repository Structure (upstream)

```
IZH-ApproxCORD/
├── hw/   — Verilog/SystemVerilog RTL sources
├── sw/   — Host-side software / drivers
└── doc/  — Documentation and figures
```

---

## Citation

```bibtex
@article{Luyen2026Izhikevich,
  author    = {Van-Vu Luyen and Thanh-Dat Nguyen and Quang-Thai Pham and
               Duy-Anh Nguyen and Khanh N. Dang and Van-Hai Pham and Dao Thanh Toan},
  title     = {Energy-Efficient Izhikevich Neuron Design Using Approximate
               CORDIC-Based Multipliers for Low-Power Neuromorphic Hardware},
  journal   = {IEEE Access},
  year      = {2026},
  doi       = {10.1109/ACCESS.2026.3662681},
  publisher = {IEEE}
}
```
