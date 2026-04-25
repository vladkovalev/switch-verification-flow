# DebugLab-Full — Fidelity Ladder + Emulation (Last Resort)

**Target:** PCIe-attached device/ASIC + x86_64 Intel/AMD management CPU.

DebugLab-Full is the high-fidelity environment that escalates through a structured model ladder:

**SystemC LT/AT → IA → CA → RTL simulation → acceleration → emulation**

The core workflow is **replay + compare**:
- replay the same golden stimulus,
- compare traces,
- capture only a minimal waveform window around **`first_bad_corr_id`** when emulation is unavoidable.

> Vendor note: DebugLab-Full is architected to be **EDA-vendor agnostic**. Commercial tool integration is **deployment-specific**; Cadence, Synopsys, and Siemens stacks are supported as examples below.

---

## Page 1 — Summary (big picture)

### Common contract
- **Common stimulus:** `transactions.parquet`
- **Common control:** Debug Portal (gRPC)
- **Compare key:** `seq + corr_id`
- **Waveforms:** backend-native exports (captured on-demand)
- **Goal:** minimize emulation usage

### What DebugLab-Full reuses
- NetLab evidence bundles
- Same Parquet schema v1
- Same corr_id semantics

### What DebugLab-Full adds
- IA/CA + deeper RTL simulation
- Acceleration options
- Native waveform databases (tool-specific)

### When to use emulation
- Only after simulation cannot resolve the bug
- Only with a narrowed corr_id window
- Only with trace-compare context attached

**Operational rule:** every stage exports a comparable `transactions.parquet`.  
Emulation is a targeted “surgical capture”, driven by trace-compare results.

---

## Page 2 — Fidelity ladder + evidence loop (corr_id driven)

Capture windows are driven by **corr_id**:
1. replay into the next engine,
2. export observed `transactions.parquet`,
3. compare to find `first_bad_corr_id`,
4. capture the smallest possible waveform window around that region.

### Fidelity ladder (conceptual)
- **SystemC LT/AT** — fast reproduction + ordering/timing relationships
- **Instruction Accurate (IA)** — ISA correctness for embedded cores
- **Cycle Accurate (CA)** — timing-critical correctness for key blocks
- **RTL Simulation** — signal-level debug + waveforms
- **Acceleration (optional)** — higher throughput than pure RTL sim
- **Emulation (last resort)** — minimal capture window using `first_bad_corr_id`

### Evidence loop
- `transactions.parquet` (stimulus + observed)
- compare → `first_bad_corr_id`

**Outcome:** emulation captures are smaller, faster, and more targeted; capacity is reserved for cases that truly require it.

---

## Page 3 — How to use (escalation playbook)

Same gRPC calls across engines. Start with the lowest-cost engine that can reproduce the issue, then escalate.

1. **Start** with a NetLab CEF bundle and pick the next backend (AT → IA → CA → RTL sim).
2. **Load stimulus** (v1 starts with DMA ignored):

```text
ReplayService.LoadStimulus(stimulus_uri=".../transactions.parquet",
  mode=STIMULUS_PLUS_CHECK,
  timing_policy=IGNORE_TIMING,
  ignore_buses=["PCIE_DMA"])

ReplayService.RunReplay()