# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

Pre-built containers ("toolboxes") for running `llama.cpp` optimally on AMD Ryzen AI Max "Strix Halo" APUs (gfx1151). Uses Toolbx (Fedora) or Distrobox (Ubuntu) on top of Podman/Docker. Leverages up to 124 GiB of unified system memory via ROCm and Vulkan backends.

Published to Docker Hub: `kyuz0/amd-strix-halo-toolboxes`

## Common Commands

### Building Containers Locally
```bash
cd toolboxes && podman build --no-cache -t llama-vulkan-radv -f Dockerfile.vulkan-radv .
```
See [docs/building.md](docs/building.md) for full build instructions per backend.

### Refreshing Toolboxes
```bash
./refresh-toolboxes.sh all   # pull latest images for all backends
```

### Benchmarking
```bash
./benchmark/run_benchmarks.sh           # single-node benchmark across toolboxes
./benchmark/run_rpc_benchmarks.sh       # distributed RPC benchmarks
python3 ./benchmark/run_mtp_bench.py    # Multi-Token Prediction benchmarking
python3 ./benchmark/generate_results_json.py  # parse results into JSON for website
```

### Distributed Inference
```bash
python3 ./scripts/run_distributed_llama.py  # TUI for cluster setup via SSH
```

## Architecture

### Backends (in `/toolboxes/`)
Each Dockerfile is a multi-stage build that compiles llama.cpp and extracts standalone binaries:

| Dockerfile | Backend | Notes |
|---|---|---|
| `Dockerfile.vulkan-radv` | Vulkan + Mesa RADV | Most compatible, auto-rebuilt |
| `Dockerfile.vulkan-amdvlk` | Vulkan + AMDVLK | Fastest, but 2 GB VRAM limit |
| `Dockerfile.rocm-6.4.4` | ROCm 6.4.4 | Stable LTS, auto-rebuilt |
| `Dockerfile.rocm-7.2.4` | ROCm 7.2.4 | Performance optimized, auto-rebuilt |
| `Dockerfile.rocm-7.2.4-rocmfpx` | ROCm + ROCmFPX | Custom FP3/FP4/FP6/FP8 quants |
| `Dockerfile.vulkan-rocmfpx` | Vulkan + ROCmFPX | ROCmFPX Vulkan variant |
| `Dockerfile.rocm7-nightlies` | ROCm nightly | Experimental |

### CI/CD (`.github/workflows/`)
- **`poll-llama-cpp.yaml`**: Runs every 4 hours, compares latest llama.cpp master SHA against stored artifact, triggers build if changed.
- **`poll-rocmfpx.yaml`**: Same pattern for the ROCmFPX upstream (offset by 30 min).
- **`build_and_publish.yml`**: Matrix build across all backends in parallel. Tags images as `<backend>_<YYYYMMDDTHHMMSS>` (immutable) + `<backend>` (channel). Includes smoke tests (version, help commands).
- **`prune-old-toolboxes.yml`**: Keeps only the 3 latest tags per backend on Docker Hub.

### Benchmark Infrastructure (`/benchmark/`)
- `run_benchmarks.sh`: Orchestrates `llama-bench` across toolboxes with OOM recovery (can restart `systemd --user` if llama-bench crashes it).
- `generate_results_json.py`: Parses raw benchmark output into structured JSON consumed by the HTML dashboards.
- Results stored in `results/`, `results-mtp/`, `results-rpc/`.

### Web Dashboards (`/docs/`)
- `index.html`, `mtp.html`, `ryzen-ai-halo.html`: Interactive benchmark viewers hosted on GitHub Pages (`strix-halo-toolboxes.com`).
- Static JSON files (`results.json`, `mtp-summary.json`) feed the dashboards.

## Critical Technical Quirks

- **Required llama.cpp flags**: Always use `-fa 1` (flash attention) and `--no-mmap` on Strix Halo to avoid memory fragmentation and crashes.
- **Unified memory**: Enabled via `LLAMA_HIP_UMA=1` and `GGML_CUDA_ENABLE_UNIFIED_MEMORY=1`.
- **Kernel boot parameters**: `amd_iommu=off amdgpu.gttsize=126976 ttm.pages_limit=32505856`
- **Kernel bugs**: Avoid kernels older than 6.18.4 and the specifically broken `linux-firmware-20251125`.
- **Device mounts**: Scripts and benchmarks use `/dev/dri` and `/dev/kfd` for GPU access. `/dev/infiniband` is auto-detected for RDMA.

## Development Guidelines

- When modifying Dockerfiles, keep builds lean — only runtime dependencies and llama.cpp binaries should be in the final image stage.
- When adding a new backend or feature, update `README.md` simultaneously.
- The VRAM estimator (`toolboxes/gguf-vram-estimator.py`) calculates memory requirements including context overhead — use it when testing new model configs.
