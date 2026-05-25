# FallDetect_Model
### Low-Power IMU-Based Fall Detection Accelerator using INT8 CNN on FPGA

🚧 **Project Status: Work In Progress**

FallDetect is an edge AI hardware project that implements real-time human fall detection using IMU sensor data and an INT8 quantized CNN accelerator.

The project targets low-power embedded platforms and focuses on transforming a trained deep learning model into deployable RTL hardware for FPGA and future ASIC implementation.

---

## Overview

The proposed architecture combines signal processing and hardware AI acceleration to enable efficient fall detection.

System pipeline:

```text
IMU Sensor
↓

M01 — Signal Conditioning
↓

M02 — Motion Event Detection
↓

M03 — DSP Trigger Controller
↓

M04 — INT8 CNN Inference Accelerator
↓

M05 — Event Reporting
↓

FPGA Validation
```

Current development focuses on building and validating the CNN accelerator pipeline.

---

# Project Objectives

- Detect human fall events from IMU data
- Reduce unnecessary inference using event-driven activation
- Execute CNN inference using INT8 arithmetic
- Deploy and validate the architecture on FPGA
- Prepare design flow for future ASIC implementation

---

# Development Workflow

## Stage 1 — AI Model Development

Purpose:

Train and optimize the fall detection model.

Pipeline:

```text
KFall Dataset
↓

Data Preprocessing

↓

Sliding Window

↓

CNN Training

↓

INT8 Quantization
```

Completed:

- ✅ KFall dataset preprocessing
- ✅ IMU feature extraction
- ✅ Sliding window generation (50 samples)
- ✅ CNN training
- ✅ Accuracy evaluation
- ✅ INT8 quantization

Generated files:

```text
falldetect_cnn.keras
falldetect_cnn_int8.tflite
mean.npy
std.npy
```

Performance:

```text
Float Accuracy ≈ 95%
INT8 Accuracy ≈ 94.79%
```

---

## Stage 2 — Hardware Mapping (RTL)

Purpose:

Convert AI model into synthesizable hardware.

Pipeline:

```text
INT8 Model
↓

Weight Export

↓

Memory Generation

↓

RTL Implementation

↓

CNN Accelerator
```

Completed:

### Weight Export

```text
conv1_weight.mem
conv1_bias.mem

conv2_weight.mem
conv2_bias.mem

dense1_weight.mem
dense1_bias.mem

dense2_weight.mem
dense2_bias.mem
```

### RTL Compute Blocks

```text
weight_rom.v
mac_int8.v
```

### CNN Layers

```text
conv1_layer.v
maxpool1d.v
conv2_layer.v

global_avg_pool.v

dense1_16.v
dense2_output.v
```

### Top Integration

```text
fall_detect_top.v
```

Current Result:

```text
RTL end-to-end inference completed
Prediction output generated successfully
```

---

## Stage 3 — FPGA Deployment

Purpose:

Validate complete system on hardware.

Pipeline:

```text
RTL
↓

Synthesis

↓

Bitstream

↓

FPGA Validation
```

Planned tasks:

- ⏳ Remove simulation-only logic
- ⏳ FPGA wrapper
- ⏳ Timing closure
- ⏳ Pin assignment
- ⏳ Bitstream generation
- ⏳ Sensor integration
- ⏳ Real-time demo

---

# Module Description

## M01 — Signal Conditioning

Function:

Preprocess raw IMU data and suppress sensor noise.

Input:

```text
Acceleration
Gyroscope
```

Output:

```text
Filtered IMU
```

Status:

```text
⏳ In Progress
```

---

## M02 — Motion Event Detection

Function:

Extract motion features and detect abnormal movement.

Example metric:

```text
Jerk = |Δax| + |Δay| + |Δaz|
```

Output:

```text
Motion score
```

Status:

```text
⏳ In Progress
```

---

## M03 — DSP Trigger Controller

Function:

Enable CNN inference only when motion exceeds threshold.

Output:

```text
cnn_enable
```

Status:

```text
⏳ In Progress
```

---

## M04 — INT8 CNN Inference Accelerator

Function:

Execute CNN inference directly in hardware.

Architecture:

```text
Input
↓

Conv1D

↓

Pooling

↓

Conv1D

↓

Global Average Pool

↓

Dense

↓

Prediction
```

Input:

```text
50 × 9
```

Output:

```text
Fall / Non-fall
```

Status:

```text
✅ Core RTL Completed (~90%)
```

---

## M05 — Event Reporting

Function:

Generate system-level output.

Output:

```text
confidence
event_flag
interrupt
```

Status:

```text
⏳ Planned
```

---

# Current Progress

| Stage | Status |
|-------|--------|
| Dataset | ✅ |
| AI Training | ✅ |
| Quantization | ✅ |
| Weight Export | ✅ |
| RTL CNN | ✅ |
| M01 | ⏳ |
| M02 | ⏳ |
| M03 | ⏳ |
| FPGA | ⏳ |

---

# Repository Structure

```text
FallDetect/

├── dataset/
├── training/
├── quantization/
├── exported_model/
├── rtl/
├── simulation/
├── fpga/
├── docs/
└── README.md
```

---

# Future Work

- Complete M01–M03 modules
- Integrate event output interface
- Deploy to FPGA
- Evaluate latency and power
- Explore ASIC implementation

---

# License

MIT License
