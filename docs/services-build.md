# Building Service Images Locally

This guide covers building the three container images used by the Quadlet service units in `services/`. These are distinct from the general-purpose toolbox images — they are runtime-only images sized for persistent `llama-server` daemons.

---

## Image Overview

| Image tag | Dockerfile | Used by | Backend |
| --- | --- | --- | --- |
| `localhost/llama-rocm-7.2.4` | `Dockerfile.rocm-7.2.4` | chat, tools (9B variant) | ROCm HIP, upstream llama.cpp |
| `localhost/llama-rocm-7.2.4-rocmfpx` | `Dockerfile.rocm-7.2.4-rocmfpx` | code | ROCm/Vulkan, charlie12345/ROCmFPX |
| `localhost/llama-rocm-7.2.4-rocmfpx1` | `Dockerfile.rocm-7.2.4-rocmfpx1` | tools (27B variant) | ROCm HIP, ciru-ai/ROCmFPX |

---

## Prerequisites

- **Podman** installed and rootless Podman configured
- **~25 GB free disk per image** during the build (builder stage is large; the final runtime image is ~8–12 GB)
- **Internet access** during the build — each Dockerfile clones llama.cpp or a fork at build time and fetches ROCm packages from repo.radeon.com
- Your user must be in the `render` and `video` groups to use the GPU

Confirm group membership before running services:

```bash
id -nG | grep -E 'render|video'
```

---

## Build Commands

All builds must be run from the `toolboxes/` directory. The Dockerfiles `COPY` files (`llama-grammar.patch`, `gguf-vram-estimator.py`) from the build context, so the working directory matters.

```bash
cd ~/code/amd-strix-halo-toolboxes/toolboxes
```

### Image 1 — ROCm 7.2.4 (upstream llama.cpp)

Used by the **chat** and **tools-9B** services.

```bash
podman build --no-cache \
  -t localhost/llama-rocm-7.2.4:latest \
  -f Dockerfile.rocm-7.2.4 \
  .
```

Expected build time: **35–50 minutes**

### Image 2 — ROCmFPX (charlie12345 fork)

Used by the **code** service. Compiles the ROCmFP4 Vulkan kernels for gfx1151.

```bash
podman build --no-cache \
  -t localhost/llama-rocm-7.2.4-rocmfpx:latest \
  -f Dockerfile.rocm-7.2.4-rocmfpx \
  .
```

Expected build time: **40–60 minutes** (FP4 kernels take longer to compile)

### Image 3 — ROCmFPX (ciru-ai fork)

Used by the **tools-27B** service. Alternative ROCmFPX implementation with HIP backend.

```bash
podman build --no-cache \
  -t localhost/llama-rocm-7.2.4-rocmfpx1:latest \
  -f Dockerfile.rocm-7.2.4-rocmfpx1 \
  .
```

Expected build time: **40–60 minutes**

> Build images sequentially rather than in parallel. Parallel builds will race for the ROCm package mirror and run three simultaneous Clang compilations, which can exhaust available RAM.

---

## Verifying a Build

After each build, confirm the key binaries are present and the GPU is accessible:

```bash
# Check llama-server is present and reports a version
podman run --rm localhost/llama-rocm-7.2.4:latest llama-server --version

# Confirm curl is present (required for Quadlet health checks)
podman run --rm localhost/llama-rocm-7.2.4:latest which curl

# Confirm rocminfo can enumerate the GPU
podman run --rm \
  --device /dev/kfd --device /dev/dri \
  --group-add keep-groups \
  --security-opt seccomp=unconfined \
  localhost/llama-rocm-7.2.4:latest \
  rocminfo | grep -A2 'Agent 2'
```

Repeat the `llama-server --version` and `which curl` checks for each image tag.

For the ROCmFPX images, also verify that MTP spec flags are available:

```bash
podman run --rm localhost/llama-rocm-7.2.4-rocmfpx:latest \
  llama-server --help | grep -i spec-type
```

If `spec-type` is absent, the `--spec-type draft-mtp` flags in the code service unit must be removed.

---

## Deploying to Quadlet

Once all images are built, copy the container files and reload:

```bash
mkdir -p ~/.config/containers/systemd

# Choose tools variant: tools-9b.container (lighter) or tools.container (27B)
cp ~/code/amd-strix-halo-toolboxes/services/llama-server-chat.container \
   ~/code/amd-strix-halo-toolboxes/services/llama-server-code.container \
   ~/code/amd-strix-halo-toolboxes/services/llama-server-tools-9b.container \
   ~/.config/containers/systemd/

# Rename the tools variant to the expected service name
mv ~/.config/containers/systemd/llama-server-tools-9b.container \
   ~/.config/containers/systemd/llama-server-tools.container

systemctl --user daemon-reload
```

Start the stack (tools starts last due to `After=` chaining; starting it pulls in the others):

```bash
systemctl --user start llama-server-tools.service
```

### Verifying services are up

```bash
systemctl --user status llama-server-chat.service
systemctl --user status llama-server-code.service
systemctl --user status llama-server-tools.service
```

Check that each loaded its model and that KV allocation is within budget:

```bash
journalctl --user -u llama-server-chat -n 200 | grep -i 'kv self'
journalctl --user -u llama-server-code -n 200 | grep -i 'kv self'
journalctl --user -u llama-server-tools -n 200 | grep -i 'kv self'
```

Sum the three `kv self` values and add the weight sizes (~21.7 + 16.9 + 9.4 GB for the 9B stack). The total must stay below ~105 GB.

---

## Rebuilding After Upstream Changes

Each image clones llama.cpp (or a fork) at build time with `--no-cache`. To pick up new commits:

```bash
podman build --no-cache -t localhost/llama-rocm-7.2.4:latest -f Dockerfile.rocm-7.2.4 .
```

Then restart the affected service:

```bash
systemctl --user restart llama-server-chat.service
```

Quadlet does not auto-pull local images — you must rebuild and restart manually.

---

## Model Files

Models must be downloaded separately to `/srv/llm/models/` before starting the services. The containers mount this directory read-only.

```bash
sudo mkdir -p /srv/llm/{models,templates}
sudo chown -R $USER /srv/llm
```

Download models with `huggingface-cli` or `hf`:

```bash
pip install -U huggingface_hub

# Chat model
hf download Jackrong/Qwopus3.6-35B-A3B-v1-MTP-GGUF \
  Qwopus3.6-35B-A3B-v1-MTP-Q4_K_M.gguf \
  --local-dir /srv/llm/models

# Code model
hf download plunderstruck/Qwopus3.6-27B-Coder-MTP-ROCmFP4-GGUF \
  Qwopus3.6-27B-Coder-MTP-ROCmFP4-STRIX-embF16-headQ6.gguf \
  --local-dir /srv/llm/models

# Tools model (9B variant)
hf download unsloth/Qwen3.5-9B-GGUF \
  Qwen3.5-9B-Q8_0.gguf \
  --local-dir /srv/llm/models

# Chat template
curl -Lo /srv/llm/templates/qwen-froggeric-v21.jinja \
  https://huggingface.co/froggeric/Qwen-Fixed-Chat-Templates/resolve/main/chat_template.jinja
```

> The Qwen3.5-9B Q8_0 GGUF repo path needs to be confirmed — search Hugging Face for the specific quantizer you prefer (Bartowski and unsloth both publish this model).
