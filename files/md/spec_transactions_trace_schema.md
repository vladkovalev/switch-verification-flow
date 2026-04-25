# Transaction Trace Schema v1 (Parquet)

## Status
- Version: v1
- Artifact: `transactions.parquet`
- Compression: **ZSTD**
- Parquet row group size: **128 MB**
- Role: **Golden stimulus** (replayable) + observation

## Goals
- Provide a backend-agnostic, filterable record of CPU↔ASIC/platform interactions.
- Enable deterministic **replay** into later fidelity stages (SystemC → IA → CA → RTL/Verilator → Cadence sim → Emulation).
- Enable cross-stage comparison by aligning on `seq` and `corr_id`.

## Key Concepts

### corr_id (Correlation ID)
- Type: uint64
- **Globally unique per run**
- Allocated at the CPU boundary layer (driver/shim issuing MMIO/interrupt programming).
- v1 rule: reuse the originating `corr_id` for subordinate events caused by the same initiating action.

### seq (Sequence Number)
- Type: int64
- Strictly increasing within a trace file.
- Defines the canonical ordering for replay and comparison.

### Timebase
Every record carries a timebase (`tb`) and time value (`t`):
- `tb`: one of `LT`, `AT`, `CYCLE`, `EMU_CYCLE`, `WALL_NS`
- `t`: int64 units in the timebase domain

**Replay ordering is by `seq` by default.** Timing is optional via replay policies.

## Parquet Schema v1

### Required columns
| Column | Type | Description |
|---|---:|---|
| `seq` | int64 | Strictly increasing record sequence number |
| `corr_id` | uint64 | Globally unique correlation ID per run |
| `tb` | string/enum | Timebase: `LT`, `AT`, `CYCLE`, `EMU_CYCLE`, `WALL_NS` |
| `t` | int64 | Timestamp in units of `tb` |
| `dir` | string/enum | Direction: `CPU_TO_SIM`, `SIM_TO_CPU` |
| `bus` | string/enum | Bus: `PCIE_MMIO`, `PCIE_DMA`, `MSIX` |
| `op` | string/enum | Operation (see below) |
| `addr` | uint64 | Address (MMIO address for `PCIE_MMIO`; 0 otherwise) |
| `len` | uint32 | Data length in bytes (0 if not applicable) |
| `data` | binary | Payload bytes (write data / read return / vector payload) |
| `status` | int32 | 0 OK; <0 error |
| `origin` | string | Emitter identifier (`cpu_driver`, `sdk`, `sim_ep`, `rtl_tb`, `emu_sb`, etc.) |
| `flags` | uint32 | Bitmask (see Flags) |

### Recommended optional columns (v1)
These columns are optional but **recommended** to maximize usefulness and reduce future schema churn:

| Column | Type | Description |
|---|---:|---|
| `bar` | uint16 | PCIe BAR number for `PCIE_MMIO` |
| `msix_vector` | uint16 | MSI-X vector number (for `MSIX`) |
| `dma_chan` | uint16 | DMA logical channel/queue |
| `dma_addr` | uint64 | DMA address (guest phys or IOVA as defined by platform) |
| `dma_len` | uint32 | DMA transfer length |
| `note` | string | Free-form annotation (diagnostic) |

## Enumerations

### bus
- `PCIE_MMIO`
- `PCIE_DMA` (optional in v1 replay; may be present but ignorable)
- `MSIX`

### op
For `PCIE_MMIO`:
- `READ`
- `WRITE`

For `MSIX`:
- `INTR_RAISE`
- `INTR_CLEAR`
- `INTR_VECTOR`

For `PCIE_DMA`:
- `DMA_SUBMIT`
- `DMA_COMPLETE`
- `DMA_READ`
- `DMA_WRITE`

## Flags (uint32 bitmask)
- bit0 `IS_STIMULUS`: record is part of golden stimulus stream
- bit1 `IS_EXPECTED`: record is expected response (e.g., read returns, interrupts)
- bit2 `NONDET`: nondeterministic event (avoid in golden traces; allowed only when unavoidable)
- bit3 `CHECKPOINT`: checkpoint marker for tooling

## Golden Stimulus Requirements (v1)
- MUST include all CPU→SIM `PCIE_MMIO/WRITE` operations needed to initialize/control the ASIC interface.
- SHOULD include:
  - MMIO reads and their returned `data` marked `IS_EXPECTED`
  - MSI-X deliveries as `MSIX/INTR_VECTOR` with `msix_vector` and `IS_EXPECTED`

### DMA
- v1 replay MUST work **without DMA support**.
- If DMA records exist, replay engines MUST support an "ignore DMA" policy.

## Replay Semantics (v1)
### Ordering
- Default: drive records in `seq` order.

### Timing policies
- `IGNORE_TIMING` (default)
- `PRESERVE_RELATIVE` (optional)
- `PRESERVE_ABSOLUTE` (optional; mostly same-backend regression)

### Checking policies
- `STIMULUS_ONLY`
- `STIMULUS_PLUS_CHECK`

## Storage Guidance
- Parquet compression: **ZSTD**
- Row group size: **128 MB**
- Partitioning: optional (by time windows) if traces become very large.

## Compatibility
- Schema is versioned by the manifest (CEF `run.json`) and by this document.
- Additive changes should prefer adding optional columns.