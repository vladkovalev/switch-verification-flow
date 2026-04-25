# switch-verification-flow
Build a vendor-agnostic verification and debug ecosystem for a PCIe-attached device/ASIC controlled by an x86_64 Intel/AMD management CPU
# NetLab + DebugLab — Replayable Evidence, Fidelity Ladder Debug

## Goal
Build a **vendor-agnostic** verification and debug ecosystem for a **PCIe-attached device/ASIC** controlled by an **x86_64 Intel/AMD management CPU**, such that teams can:

- reproduce issues quickly in integration/topology environments,
- capture evidence once and **replay it everywhere**,
- escalate to higher fidelity (including commercial emulation) **only when necessary**,
- keep results comparable across engines using a stable evidence and control contract.

In short: **simulation-first**, with **trace-driven escalation** and **minimal waveform capture**.

---

## What we provide (the solution)
This project defines a tiered workflow and a set of artifacts/contracts that connect three subsystems:

### 1) NetLab (GNS3)
**Purpose:** topology + integration + fast reproduction  
**What it does:**
- runs multi-node lab topologies in **GNS3**
- uses a **CPU VM** for the management/control-plane software
- uses a **SIM node** (e.g., SystemC LT/AT) to model the device/ASIC behavior

**Primary output:** a replayable “golden stimulus” evidence bundle:
- `transactions.parquet` — ordered transaction trace (compressed, efficient)
- `run.json` — run manifest + metadata/log pointers
- optional semantic traces (e.g., RTEdbg)

### 2) DebugLab-Lite (open-source deep debug)
**Purpose:** waveform + semantic debug with open tooling  
**What it does:**
- replays `transactions.parquet` into an open RTL backend (e.g., Verilator)
- produces:
  - `waves.fst` waveforms (GTKWave-friendly)
  - observed `transactions.parquet` (cycle timebase)
  - semantic traces (e.g., RTEdbg) for intent-level correlation

### 3) DebugLab-Full (internal fidelity ladder)
**Purpose:** maximum fidelity, used as a last resort  
**What it does:**
- escalates through a structured ladder such as:
  - SystemC LT/AT → IA → CA → RTL simulation → acceleration → emulation
- uses the same replay+compare concept to minimize expensive runs
- captures waveforms **only around the first divergence**, not continuously

**EDA vendor integration is deployment-specific**, but the workflow supports internal stacks such as:
- Cadence (e.g., Stratus/VSP/Helium/Xcelium)
- Synopsys (e.g., Platform Architect/Virtualizer/VCS)
- Siemens (e.g., Catapult/Questa/Veloce)

---

## The core contracts (what makes escalation possible)

### A) Debug Portal (gRPC)
A uniform control plane for:
- capability discovery
- start/stop trace capture
- load/replay stimulus
- stream progress/events
- request targeted waveform capture/export (where supported)

### B) Replayable evidence: `transactions.parquet`
A single, portable evidence format used across subsystems:
- compressed and scalable (Parquet + ZSTD)
- ordered by `seq`
- correlated by `corr_id` (globally unique per run)

**Key workflow mechanic:** compare traces by `seq + corr_id` to find:
- `first_bad_corr_id`

Then capture only a minimal waveform window around that region.

---

## Why split the system
A monolithic environment forces all users to pay the cost and friction of the highest-fidelity tooling on every iteration.

Splitting into NetLab / Lite / Full optimizes for:
- **velocity** (fast reproduction and evidence capture),
- **depth** (waveform-level or emulation-level debug when needed),
- **cost control** (emulation reserved for cases that truly require it),
- **reproducibility** (same stimulus replayed across engines).

---

## How to navigate the docs

### HTML pages
- `files/html/Overview.html`
- `files/html/NetLab.html`
- `files/html/DebugLab-Lite.html`
- `files/html/DebugLab-Full.html`

### Markdown sources
- `files/md/Overview.md`
- `files/md/NetLab.md`
- `files/md/DebugLab-Lite.md`
- `files/md/DebugLab-Full.md`

---

## Summary
**Capture once. Replay everywhere. Compare by corr_id. Escalate only when required.**
This is the design principle behind NetLab + DebugLab.
