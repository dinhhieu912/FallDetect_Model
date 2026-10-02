# Fall Detection INT8 IP

A low-power, DSP-gated fall detection IP. A lightweight DSP trigger monitors acceleration continuously and wakes a bit-accurate INT8 CNN accelerator only when a suspicious motion is detected. The CNN is frozen from `falldetect_small_int8.tflite` and verified byte-for-byte against the LiteRT reference kernels.

## Architecture

```text
IMU sample (50 Hz-100 Hz)
  ├─ accel_x/y/z (signed16 Q5.10) ─> IIR LPF ─> accel/jerk energy ─> trigger FSM
  └─ 9 x INT8 CNN features ────────> 50x9 circular window buffer
                                              │ (trigger fires)
                                              ▼
                              Conv1 -> Pool1 -> Conv2 -> Pool2 -> GAP -> Dense1 -> Dense2
                                              ▼
                                   logit + fall flag -> APB3 registers + IRQ
```

Frozen network:

```text
INT8 [50,9]
  -> Conv1 K5 SAME, 16, ReLU     [50,16]
  -> MaxPool K2 S2 VALID         [25,16]
  -> Conv2 K3 SAME, 32, ReLU     [25,32]
  -> MaxPool K2 S2 VALID         [12,32]
  -> Global Average Pool         [32]
  -> Dense 16, ReLU              [16]
  -> Dense 1                     [1 logit]
```

The hardware outputs the pre-sigmoid logit; a fall is declared when the real logit is greater than 0 (quantized Dense2 zero point is -14). No hardware sigmoid is needed.

Key design points:

- Square-root-free trigger: energies are sums of squares; the LPF coefficient is `2^-LPF_SHIFT`, so it maps to adders and shifts.
- Single-buffer policy: acquisition pauses while the window drains into the CNN. CNN latency is under 1 ms at 50 MHz, shorter than typical IMU sample intervals. A ping-pong buffer is the upgrade path if acquisition must never pause.
- Technology-neutral: synchronous ROM wrapper, packed 4-lane weight ROMs, no vendor primitives.

## Repository layout

| Path | Content |
|---|---|
| `rtl/` | SystemVerilog sources and `filelist.f` |
| `mem/` | Weights, biases, Q31 multipliers and shifts (`.mem`) |
| `model/` | Frozen TFLite model, manifest (SHA-256), golden-model and calibration scripts |
| `verification/` | Self-checking testbenches, vectors, regression logs and reports |
| `architecture/` | Per-module design notes, APB register map, sensor contract |
| `constraints/` | 50 MHz SDC / XDC |
| `scripts/` | Icarus, Yosys, NanGate45 and Vivado flows (PowerShell/Tcl) |
| `implementation/` | Synthesis flow notes |

Main RTL modules: `requantize_int8`, `conv1d_engine`, `maxpool1d_stream`, `global_avg_pool_int8`, `dense_engine`, `fall_cnn_int8_top`, `lpf_iir3`, `jerk_energy3`, `dsp_trigger_fsm`, `window_buffer_50x9`, `fall_detect_app_top`, and the synthesis top `fall_detect_apb_top`.

## Interface

Top: `fall_detect_apb_top` (parameters `LPF_SHIFT=4`, `ENERGY_WIDTH=40`).

| Port | Description |
|---|---|
| `PCLK`, `PRESETn`, `PSEL`, `PENABLE`, `PWRITE`, `PADDR[7:0]`, `PWDATA`, `PRDATA`, `PREADY`, `PSLVERR` | APB3 slave |
| `sample_valid` / `sample_ready` | Sample handshake |
| `accel_x/y/z` | Signed 16-bit Q5.10 acceleration (1024 counts/g) for the DSP trigger |
| `cnn_features_int8[71:0]` | Nine INT8 model inputs, channel `c` at bits `c*8 +: 8` |
| `irq` | Sticky event interrupt |

Channel order: AccX, AccY, AccZ, GyrX, GyrY, GyrZ, EulerX, EulerY, EulerZ.

CNN feature quantization (done outside the IP):

```text
normalized = (physical_value - mean[c]) / std[c]
int8 = clamp(round(normalized / 0.08433684706687927) - 22, -128, 127)
```

### APB3 register map

All registers are 32-bit and complete with zero wait states. Invalid or unaligned access asserts `PSLVERR`.

| Address | Name | Access | Description |
|---:|---|---|---|
| 0x00 | ID | RO | `0x46414C4C` ("FALL") |
| 0x04 | CONTROL | RW | bit 0 enable, bit 1 IRQ enable |
| 0x08 | STATUS | RO | bit 0 DSP armed, bit 1 CNN active, bit 2 event pending, bit 3 sample ready |
| 0x0C / 0x10 | ACCEL_THRESHOLD_LO / HI | RW | Acceleration energy threshold (40 bit) |
| 0x14 / 0x18 | JERK_THRESHOLD_LO / HI | RW | Jerk energy threshold (40 bit) |
| 0x1C | TRIGGER_CFG | RW | Consecutive suspicious samples, bits 7:0 |
| 0x20 | EVENT | RO | Logit bits 7:0, fall bit 8 |
| 0x24 | IRQ_STATUS | RO/W1C | Event pending in bit 0; write 1 to clear |

Reset defaults: IP disabled, IRQ enabled, acceleration threshold at maximum (jerk-only trigger), jerk threshold 14,325, consecutive count 3. Events are captured in sticky registers and the pipeline is released immediately, so monitoring resumes before firmware services the IRQ.

## DSP trigger calibration (KFall)

The trigger was calibrated by running the RTL integer arithmetic over the full KFall dataset: 5,075 trials, 3,995,100 samples at 100 Hz. Acceleration never clipped in signed16 Q5.10.

Default operating point: `LPF_SHIFT=4`, jerk threshold 14,325, 3 consecutive samples.

| Target ADL trigger rate | Measured ADL trigger rate | Fall trigger recall |
|---:|---:|---:|
| 10% | 9.97% | 87.51% |
| 20% (default) | 19.97% | 97.10% |
| 50% | 49.98% | 100.00% |

The default prioritizes fall recall while keeping the CNN asleep in about 80% of ADL trials. The CNN then rejects triggered ADL windows. Thresholds can be changed at runtime through APB without RTL changes. These are trial-level trigger statistics, not end-to-end model sensitivity/specificity.

## Verification

Reference: LiteRT reference kernels (`BUILTIN_REF`), model SHA-256 `FE44C82B...FBF8AE` locked in `model/model_manifest.json`. Every intermediate tensor (Conv1, Pool1, Conv2, Pool2, GAP, Dense1, Dense2) matches byte-for-byte.

Icarus Verilog 12 regression: 22/22 baseline testbenches pass, with no compile failures, X/Z, or `$readmemh` failures.

| Area | Result |
|---|---|
| Requantization (256 vectors) | PASS |
| Conv1 / Pool1 / Conv2 / Pool2 / GAP / Dense1 / Dense2 full tensors | PASS (800 / 400 / 800 / 384 / 32 / 16 / 1) |
| Full CNN | PASS, logit byte 11, fall decision |
| DSP trigger -> window -> CNN -> event | PASS |
| Circular buffer ordering after pointer wrap | PASS, 450/450 bytes |
| Trigger FSM (consecutive count, disarm, rearm) | PASS |
| APB config, IRQ, W1C clear | PASS |
| Quiet motion keeps CNN asleep | PASS |
| Triggered non-fall | PASS, logit -35, fall = 0 |
| Reset during inference, then recovery | PASS |
| Two consecutive inferences | PASS, 2/2 |
| Real KFall (33 SA12 windows) | PASS, 33/33 RTL logits match golden; 31/33 labels correct |

Latency (50 MHz, with output backpressure): Conv1 19,598 cycles; full CNN 39,597 cycles (about 0.79 ms); trigger to held event 40,051 cycles.

Testbenches terminate with `$fatal` if vector files are missing, so a failed `$readmemh` cannot produce a false pass. Legacy testbenches in `verification/legacy_logs/` are smoke-only and do not prove equivalence with TFLite. See `verification/FULL_REGRESSION.md` and `verification/VERIFICATION_MATRIX.md`.

## Quick start

Regenerate golden artifacts (requires `numpy`, `tflite`, `flatbuffers`, `ai-edge-litert`):

```powershell
python model/build_model_artifacts.py --model model/falldetect_small_int8.tflite --output .
```

Run the full RTL regression (Icarus Verilog):

```powershell
powershell -ExecutionPolicy Bypass -File scripts/run_iverilog.ps1 -IverilogRoot "D:\Icarus\iverilog"
```

Run a single block, for example requantization:

```powershell
iverilog -g2012 -s tb_requantize_int8 -o sim_requant rtl/requantize_int8.sv verification/tb_requantize_int8.sv
vvp sim_requant
```

Waveform demo: `scripts/run_waveform_demo.ps1` (see `verification/WAVEFORM_DEMO.md`).

## Implementation

Synthesis top: `fall_detect_apb_top`, target clock 50 MHz (20 ns).

```powershell
# Generic Yosys
powershell -ExecutionPolicy Bypass -File scripts/run_yosys_generic.ps1 -YosysExe yosys

# NanGate45
powershell -ExecutionPolicy Bypass -File scripts/run_nangate45.ps1 -YosysExe yosys -Liberty <path>\NanGate_45nm_OCL_typical.lib

# Vivado FPGA (change the part to your device)
vivado -mode batch -source scripts/run_vivado_impl.tcl -tclargs xc7a35tcpg236-1
```

Constraints are in `constraints/fall_detect_50mhz.sdc` and `.xdc`. No bitstream is generated because board pinout is not defined.

## Known limitations

- No synthesis, timing, or power results are reported yet; none were run for this baseline.
- Requantization supports real multipliers strictly between 0 and 1 (true for all multipliers in this model). Supporting larger multipliers needs an interface revision with new overflow tests.
- The raw-to-INT8 preprocessing is not in RTL. It stays outside the IP until the physical sensor unit, range and fixed-point format are frozen. A different sensor scale requires threshold rescaling.
- Single window buffer: sampling pauses during CNN inference.
- A formal subject-independent CNN evaluation split is a model-evaluation artifact, not part of RTL equivalence.
