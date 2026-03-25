<p align="center">
  <img src="./assets/alsdjlfkweoincoweubhowvubaelr.png" alt="MLM Memory Engine Banner">
</p>

<p align="center">
  <img alt="Cross Platform" src="https://img.shields.io/badge/CROSS_PLATFORM-Linux%20%7C%20macOS%20%7C%20Windows-00ff9f?style=for-the-badge&labelColor=0a0a0f">
  <img alt="Ollama" src="https://img.shields.io/badge/OLLAMA-SCOPED_RUNTIME-ff006e?style=for-the-badge&logo=ollama&logoColor=ffffff&labelColor=0a0a0f">
  <img alt="Headroom" src="https://img.shields.io/badge/TARGET-2x%E2%80%933x_HEADROOM-00ff9f?style=for-the-badge&labelColor=0a0a0f">
</p>

# MLM Memory Engine

Cross-platform scoped memory optimization for Ollama.

- `mlm-memory` handles private runtime orchestration (`analyze`, `setup`, `run`, `status`)
- `mlm-memory-extended` handles planning/observability (`preflight`, `recommend`, `context-fit`, `monitor`)

## Highlights

### 2026-03-19 (core rewrite)

- Replaced legacy Bash flow with Python CLI.
- Added Linux, macOS, and Windows 10+ support.
- Moved from global tuning to process-scoped private loopback runtime.
- Added adaptive hardware profiling and artifact generation.

### 2026-03-25 (extension)

- Added `mlm-memory-extended` with Layer 4–7 analysis commands.
- Added architecture-aware model recommendation (MLA / MoE / Hybrid / Transformer).
- Added context-fit optimizer and live memory pressure monitor.

## What This Project Optimizes

```text
Goal: improve practical model fit headroom (~2x to 3x in many real workloads)
Method: lower runtime pressure (ctx/batch/concurrency) + KV quantization + scoped execution
```

This is heuristic tuning, not a hard guarantee. Final fit depends on quantization, prompt length, adapters, and host load.

## Requirements

- Python 3.9+
- Ollama installed and resolvable as `ollama` (or set `OLLAMA_BIN`)

## Quick Start

### macOS / Linux

```bash
cd /path/to/mlm
chmod +x ./mlm-memory ./mlm-memory-extended

# 1) Baseline host/profile artifacts
./mlm-memory analyze

# 2) Evaluate and validate model fit
./mlm-memory-extended recommend
./mlm-memory-extended preflight --model deepseek-r1:14b
./mlm-memory-extended context-fit --model deepseek-r1:14b --target 0.85

# 3) Create scoped alias and run
./mlm-memory setup --model deepseek-r1:14b
./mlm-memory run --model deepseek-r1:14b

# 4) Optional live monitor in another terminal
./mlm-memory-extended monitor
```

### Windows 10+

```powershell
cd C:\path\to\mlm

python .\mlm-memory analyze
python .\mlm-memory-extended recommend
python .\mlm-memory-extended preflight --model deepseek-r1:14b
python .\mlm-memory-extended context-fit --model deepseek-r1:14b --target 0.85
python .\mlm-memory setup --model deepseek-r1:14b
python .\mlm-memory run --model deepseek-r1:14b
```

## Commands

### Base CLI (`mlm-memory`)

```text
mlm-memory analyze
mlm-memory setup [--model MODEL]
mlm-memory run [--model MODEL] [MODEL] [-- <ollama run args...>]
mlm-memory status
mlm-memory help
```

### Extension CLI (`mlm-memory-extended`)

```text
mlm-memory-extended preflight --model MODEL [--params-b FLOAT] [--quant QUANT]
mlm-memory-extended recommend
mlm-memory-extended context-fit --model MODEL [--params-b FLOAT] [--quant QUANT] [--target FLOAT]
mlm-memory-extended monitor [--interval FLOAT]
```

## Adaptive Profile Logic (`mlm-memory`)

| Profile | Condition | num_ctx | num_batch |
|---|---|---:|---:|
| survival | total RAM < 16 GiB | 1024 | 32 |
| aggressive | 16 GiB <= total RAM < 32 GiB | 1536 | 64 |
| balanced-aggressive | total RAM >= 32 GiB | 2048 | 128 |
| cpu-tight | no GPU and total RAM < 24 GiB (override) | capped at 1024 | capped at 24 |
| unified-large | unified memory and total RAM >= 64 GiB (override) | 3072 | 192 |

Notes:
- `num_thread` is derived from logical CPU count.
- `num_gpu=999` is used on unified-memory hosts.

## Scoped Runtime Tuning (`mlm-memory run`)

Applied only to the private server process:

- `OLLAMA_HOST=<random 127.0.0.1:port>`
- `OLLAMA_KV_CACHE_TYPE=q4_0`
- `OLLAMA_FLASH_ATTENTION=1`
- `OLLAMA_MAX_LOADED_MODELS=1`
- `OLLAMA_KEEP_ALIVE=0`
- `OLLAMA_NUM_PARALLEL=1`
- `OLLAMA_MAX_QUEUE=1`
- `OLLAMA_CONTEXT_LENGTH=<derived num_ctx>`
- `OLLAMA_NO_CLOUD=1` (if unset)

Generated Modelfile params include `num_ctx`, `num_batch`, `num_thread`, and `num_gpu` (when applicable).

## Extension Behavior (`mlm-memory-extended`)

### `preflight`

- Computes projected weights + KV memory.
- Finds an optimal context via binary search (internal fill target `0.88`).
- Verdict bands: `<75%` comfortable, `75%–92%` tight, `>=92%` critical.

### `recommend`

- Ranks built-in catalog by budget fit.
- Tier thresholds: fit `<=80%`, tight `<=105%`, otherwise too large.

### `context-fit`

- Prints context-vs-memory table.
- Recommends `PARAMETER num_ctx` for your target fill (`--target`, default `0.85`).
- MLA entries include conservative-estimate note and adjusted context hint.

### `monitor`

- macOS uses `vm_stat`; Linux uses `/proc/meminfo`.
- Poll interval default is `1.0s`.
- Pressure bands: normal `<65%`, elevated `65%–82%`, high `82%–92%`, critical `>92%`.

## Artifacts

Stored in `~/.ollama/mlm-memory_modelfiles`:

- `analysis.json`
- `manifest.json`
- `<model-hash>.Modelfile`
- `last-run-server.log`

## Validation

```bash
python3 -m py_compile mlm-memory
python3 -m py_compile mlm-memory-extended
./mlm-memory --help
./mlm-memory-extended --help
./mlm-memory analyze
./mlm-memory status
```

## Docs Sync (HTML)

`mlm-memory.html` is generated from `README.md` + `mlm-memory`:

```bash
./sync-docs
```

## Migration Notes

- Old `*-mlm` tags and new `mlm-memory-*:scoped` tags can coexist.
- Similar sizes in `ollama list` do not always imply duplicate blob storage.

Clean scoped aliases only:

```bash
ollama list | awk 'NR>1 {print $1}' | rg '^mlm-memory-' | while read -r m; do ollama rm "$m"; done
```

Optional legacy cleanup:

```bash
ollama rm richardyoung/qwen3-14b-abliterated-mlm:latest
```

## Rollback

```bash
rm -rf ~/.ollama/mlm-memory_modelfiles
```

## Known Limits

- Layer 4 in extension is estimator/recommendation logic, not runtime KV eviction.
- True paged attention requires runtime support outside this project.
- MLA fit outputs are intentionally conservative.
