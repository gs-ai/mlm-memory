<p align="center">
  <img src="./assets/mlm-memory-banner.png" alt="MLM Memory Engine Preview">
</p>

<p align="center">
  <img alt="Cross Platform" src="https://img.shields.io/badge/CROSS_PLATFORM-Linux%20%7C%20macOS%20%7C%20Windows-00ff9f?style=for-the-badge&labelColor=0a0a0f">
  <img alt="Ollama" src="https://img.shields.io/badge/OLLAMA-SCOPED_RUNTIME-ff006e?style=for-the-badge&logo=ollama&logoColor=ffffff&labelColor=0a0a0f">
  <img alt="Headroom" src="https://img.shields.io/badge/TARGET-2x%E2%80%933x_HEADROOM-00ff9f?style=for-the-badge&labelColor=0a0a0f">
</p>

# ∞ MLM-MEMORY ∞

`ACID // PLASMA // VOID`

Cross-platform scoped memory optimization for Ollama.

`mlm-memory` analyzes your machine (CPU, RAM, GPU where available), derives an aggressive memory profile, and launches models through a private scoped Ollama runtime so other Ollama usage on the system stays unchanged.

## WHAT CHANGED (2026-03-19)

- Replaced the old Bash workflow with a Python CLI (`analyze`, `setup`, `run`, `status`, `help`).
- Added cross-platform support for Linux, macOS, and Windows 10+ (no Docker).
- Switched from global/system-level tuning to scoped/private runtime tuning only.
- `run` now starts a private loopback Ollama server for isolation and prints a shutdown report on exit.
- Added hardware analysis and adaptive profile selection instead of fixed Apple-only assumptions.
- Added generated artifacts: `analysis.json`, `manifest.json`, model-specific Modelfiles, and run logs.

## MIGRATION NOTES (FROM OLDER MLM-MEMORY)

- Old `*-mlm` tags and new `mlm-memory-*:scoped` tags can coexist.
- `ollama list` may show similar sizes for related tags; this does not always mean duplicate blob storage.
- If you want to clean old scoped aliases only:

```bash
ollama list | awk 'NR>1 {print $1}' | rg '^mlm-memory-' | while read -r m; do ollama rm "$m"; done
```

- If you want to remove legacy custom tags (optional):

```bash
ollama rm richardyoung/qwen3-14b-abliterated-mlm:latest
```

## TARGET

```text
Goal: increase practical model fit headroom by ~2x to 3x
Method: tighter context/batch/concurrency + KV cache quantization + scoped runtime isolation
```

Important: this is heuristic tuning, not a hard guarantee. Final fit still depends on model quantization, prompt length, adapters, and Ollama build behavior.

## WHAT IT DOES

1. Host analysis (Linux, macOS, Windows 10+)
2. Memory profile derivation from detected hardware
3. Scoped Modelfile generation for your selected base model
4. Private loopback `ollama serve` launch with scoped env only
5. Scoped alias build + run
6. Shutdown/status report with active models and Ollama processes

## WHAT IT DOES NOT DO

- No global `launchctl` edits
- No shell profile edits (`.zshrc`, `.bashrc`, etc.)
- No Docker
- No universal override of all Ollama models

## REQUIREMENTS

- Python 3.9+
- Ollama installed and available as `ollama` (or set `OLLAMA_BIN`)

## QUICK START

### macOS / Linux

```bash
cd /path/to/mlm
chmod +x ./mlm-memory
./mlm-memory analyze
./mlm-memory setup
./mlm-memory run
```

### Windows 10+

```powershell
cd C:\path\to\mlm
python .\mlm-memory analyze
python .\mlm-memory setup
python .\mlm-memory run
```

## COMMANDS

```text
mlm-memory analyze
mlm-memory setup [--model MODEL]
mlm-memory run [--model MODEL] [MODEL] [-- <ollama run args...>]
mlm-memory status
mlm-memory help
```

Examples:

```bash
./mlm-memory setup --model qwen2.5:14b
./mlm-memory run --model qwen2.5:14b
./mlm-memory run llama3.1:8b -- --verbose
./mlm-memory status
```

## SCOPED TUNING USED

The run path applies process-local Ollama settings such as:

- `OLLAMA_KV_CACHE_TYPE=q4_0`
- `OLLAMA_FLASH_ATTENTION=1`
- `OLLAMA_MAX_LOADED_MODELS=1`
- `OLLAMA_KEEP_ALIVE=0`
- `OLLAMA_NUM_PARALLEL=1`
- `OLLAMA_MAX_QUEUE=1`

And per-model Modelfile parameters such as:

- `num_ctx` (reduced)
- `num_batch` (reduced)
- `num_thread` (auto-sized)
- `num_gpu` (set when appropriate, e.g. unified-memory Apple Silicon)

## ARTIFACTS

Created under `~/.ollama/mlm-memory_modelfiles`:

- `analysis.json`
- `manifest.json`
- `<model-hash>.Modelfile`
- `last-run-server.log`

## SHUTDOWN CONFIRMATION

After `run` exits (including Ctrl+C), the tool reports:

- private scoped server shutdown status
- scoped server model state before shutdown
- currently running Ollama processes with PID and cwd when available
- default server model state if reachable

## VALIDATION

```bash
python3 -m py_compile mlm-memory
./mlm-memory --help
./mlm-memory analyze
./mlm-memory status
```

## ROLLBACK

```bash
rm -rf ~/.ollama/mlm-memory_modelfiles
```
