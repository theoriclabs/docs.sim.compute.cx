# tsim Agent Skill — Virtual GPUs for Frontier Kernels

Use **tsim** (`https://api.sim.compute.cx`) to compile, functionally verify, time, and stall-profile CUDA C++, PTX, Triton, and ELF64 `.cubin` kernels on calibrated **NVIDIA H200 (`sm_90a`), B200 (`sm_100a`), and B300 (`sm_103a`)** virtual silicon—using CPU cores only, with zero physical GPU spend during search.

## Prerequisites

- Environment variables:
  - `TSIM_API_URL` (default `https://api.sim.compute.cx`)
  - `TSIM_API_KEY` (personal `tsim_live_...` token minted at `https://sim.compute.cx/keys`)
- Verify auth before starting a campaign:
  ```bash
  curl -fsS "${TSIM_API_URL:-https://api.sim.compute.cx}/v2/auth/whoami" \
    -H "Authorization: Bearer $TSIM_API_KEY"
  ```

## Calibrated Targets

| Target ID | Arch | Locked Clock | Status |
| --- | --- | --- | --- |
| `nvidia/h200_sxm` | `sm_90a` | `1980 MHz` | Calibrated (`ρ=0.9827`, `20/20` exact) |
| `nvidia/b200_sxm` | `sm_100a` | `1965 MHz` | Calibrated (`ρ=0.9802`, `19/19` exact) |
| `nvidia/b300_sxm` | `sm_103a` | `2032 MHz` | Calibrated (`ρ=0.9662`, `18/18` exact) |

## Core API Workflow

1. **Upload concrete input & expected-output tensors once (`POST /v2/blobs`):**
   - Check existence first with `HEAD /v2/blobs/sha256:<hex>`.
   - Upload raw bytes with `Content-Type: application/octet-stream`.
   - For large dense read-only weight matrices whose values do not control branches/addresses, omit `blob_ref` on `access: "ro"` buffers to avoid payload overhead (1-bit taint will return `CONCRETE_INPUTS_REQUIRED` if concrete data is needed).
2. **Compile candidate source on CPU (`POST /v2/compile`):**
   - Payload: `{"language": "cuda"|"ptx"|"triton", "target": "nvidia/b300_sxm", "entry": "<sym>", "flags": ["-O3"], "source": "..."}`
   - Returns `blob_ref`, `regs_per_thread`, `smem_bytes`, `ptxas_log`, `cached`.
   - If `status == "compile_error"`, inspect `error` and `ptxas_log` and fix source before retrying.
3. **Verify, time & profile (`POST /v2/evaluate`):**
   - Always pass `protocol: {"mode": "verify_and_time", "cache_state": "cold_l2", "replay_count": 1}` and `verification: {"comparator": "exact_bytes"|"allclose", "expected_outputs": {...}}`.
   - Pass `diagnostics: {"level": "instruction_hotspots", "top_k": 5}` to read `summary_counters` (`stall_mem_wait`, `stall_exec_dep`, `stall_barrier`, `stall_pipe_busy`) and top stalling PCs.
   - Never accept a faster timing unless `case.status == "ok"` and `case.correctness.passed == true`.
4. **Concurrency & retries:**
   - Respect `HTTP 429` + `Retry-After` when submitting parallel candidates.
   - Do not retry `TimeoutError` blindly—reduce grid size or eliminate register spills/infinite loops.
5. **Final physical verification:**
   - Once CPU search converges, verify only the winning `.cubin` on a physical GPU (e.g. via `compute` CLI or Modal).
