# Overview — NetLab + DebugLab (Lite/Full)

We build a tiered verification and debug ecosystem for a **PCIe-attached device/ASIC** with an **x86_64 Intel/AMD management CPU**:
- **NetLab** for topology/integration and fast reproduction
- **DebugLab-Lite** for open-source waveform debug (Verilator + RTEdbg)
- **DebugLab-Full** for the internal fidelity ladder ending with emulation as a last resort

---

## At a glance (shared contracts)
- **Debug Portal:** gRPC
- **Golden stimulus:** `transactions.parquet`
- **Compression:** ZSTD
- **Row groups:** 128MB
- **corr_id:** globally unique per run
- **PCIe v1 focus:** MMIO + MSI-X (DMA optional)

> **Core idea:** capture once (in NetLab), replay everywhere. Compare traces by `seq + corr_id`.  
> Escalate fidelity only when required, and capture waveforms only around the first divergence.

---

## Goal & why split the system

### Goal
- Fast iteration in simulation-first environments
- Replayable evidence across all stages
- Emulation only as a last resort

### Why split
- NetLab scales to many nodes and scenarios
- Lite provides waveforms with open tools
- Full provides the internal fidelity ladder

### What makes it work
- gRPC Debug Portal (uniform control plane)
- Parquet golden stimulus (uniform evidence)
- corr_id anchors (uniform correlation)

> **EDA vendor neutrality:** DebugLab-Full is designed to integrate with internal commercial stacks as needed.  
> Deployments may use **Cadence**, **Synopsys**, and/or **Siemens** tooling (deployment-dependent).

---

## Three subsystems (roles & outputs)

### NetLab (GNS3)
- Topology + lifecycle (GNS3)
- CPU VM (Ubuntu + SDK/SAI)
- SIM node (SystemC LT/AT)
- **Output:** golden stimulus `transactions.parquet` + `run.json`

### DebugLab-Lite (External)
- Replay stimulus into Verilator
- **Waveforms:** FST (GTKWave)
- **Semantics:** RTEdbg traces (+ optional VCD markers)
- **Output:** FST + observed Parquet

### DebugLab-Full (Internal)
- Fidelity ladder: LT/AT → IA → CA → RTL sim
- Acceleration (optional)
- Emulation (last resort)
- **Output:** comparable Parquet + native wave DB on demand

---

## Workflow (conceptual)
1) **NetLab capture** → `run.json` + `transactions.parquet` (+ optional RTEdbg)  
2) **Replay** in DebugLab-Lite and/or DebugLab-Full  
3) **Compare** by `seq + corr_id` → find `first_bad_corr_id`  
4) **Minimal wave capture** around the first divergence
