# Debug Portal gRPC API v1 (Conceptual Spec)

## Purpose
Provide a uniform control/debug interface across backends:
- SystemC LT/AT
- Instruction-accurate (IA)
- Cycle-accurate (CA)
- RTL simulation (incl. Verilator)
- Cadence simulation & emulation (internal)

The portal is used by:
- NetLab (GNS3-facing workflows)
- DebugLab-Lite (open-source workflows)
- DebugLab-Full (internal workflows)

## Core Principles
- Capability negotiation: clients must query capabilities and adapt.
- Streaming-friendly: long-running operations stream progress/events.
- Artifact-oriented: traces/waves exported as files referenced by URI/path.
- Replay-first: load & replay golden stimulus traces.

## Services (v1)

### 1) CapabilitiesService (required)
**GetCapabilities()**
Returns:
- backend_id, backend_type (`systemc_lt`, `systemc_at`, `ia`, `ca`, `rtl`, `verilator`, `cadence_sim`, `cadence_emu`)
- supported_timebases: `LT`, `AT`, `CYCLE`, `EMU_CYCLE`, `WALL_NS`
- supports_replay (bool)
- supports_waveforms (bool)
- supports_step (bool)
- supported_trace_buses (list): `PCIE_MMIO`, `MSIX`, `PCIE_DMA`
- supported_artifacts (list): `transactions.parquet`, `waves.fst`, `cadence_db`, `rtedbg.bin`

### 2) RunControlService (required)
- **Reset(ResetRequest)**: reset DUT/sim
- **Run(RunRequest)**: run/free-run until halted/timeout
- **Halt(HaltRequest)**: halt/stop
- **Step(StepRequest)**: optional; step by N cycles/instructions if supported
- **GetStatus(StatusRequest)**: running/halted, current timebase+time, last error

### 3) TraceService (required)
- **StartTrace(StartTraceRequest)**
  - output_dir (string)
  - trace_buses (repeated): e.g., `PCIE_MMIO`, `MSIX`
  - filter rules: corr_id ranges, address ranges, origins
  - parquet settings: compression=`ZSTD`, row_group_size_mb=`128`
- **StopTrace(StopTraceRequest)**
Returns:
- artifact URIs (at minimum `transactions.parquet`)

### 3a) TraceService.StreamEvents (optional, recommended)
- **StreamEvents(StreamEventsRequest)** (server-streaming)
Event messages may include:
- progress updates
- warnings/errors
- markers: `(seq, corr_id, tb, t, message)`
Used for UI/CLI live feedback and for quickly locating first divergence.

### 4) ReplayService (required if supports_replay=true)
- **LoadStimulus(LoadStimulusRequest)**
  - stimulus_uri (e.g., path to `transactions.parquet`)
  - mode: `STIMULUS_ONLY` / `STIMULUS_PLUS_CHECK`
  - timing_policy: `IGNORE_TIMING` / `PRESERVE_RELATIVE` / `PRESERVE_ABSOLUTE`
  - ignore_buses: e.g. ignore `PCIE_DMA` for v1
- **RunReplay(RunReplayRequest)**: start replay
- **GetReplayResult(GetReplayResultRequest)**
Returns:
- pass/fail
- first_bad_seq (optional)
- first_bad_corr_id (optional)
- summary string
- artifact URIs for produced traces

### 5) WaveformService (optional, recommended)
Exposed only if supports_waveforms=true.

- **StartCapture(StartCaptureRequest)**
  - output_dir
  - format preference: `FST` (Verilator) or backend-native
  - signal_profile: `minimal`, `pcie`, `scheduler`, `full` (backend-defined)
  - window selection:
    - by corr_id range (preferred)
    - by time range (tb+t)
- **StopCapture(StopCaptureRequest)**
- **ExportWaveforms(ExportWaveformsRequest)**
Returns:
- waveform artifact URI(s): `waves.fst` or Cadence emulation-native DB references
- `wave_index.json` mapping corr_id markers to time windows (recommended)

## Error Handling
- Methods return a structured status with:
  - code
  - message
  - optional remediation hint (e.g., "waveforms unsupported in systemc_lt backend")

## Security / Deployment (out of scope v1)
- gRPC channel security (mTLS) is deployment-specific.
- Authentication/authorization is deployment-specific.