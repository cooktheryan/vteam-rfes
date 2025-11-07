# Container CLI Contract

**Feature**: 001-ubi-vllm-container
**Purpose**: Define the command-line interface contract for building, running, and managing the UBI vLLM container

---

## Container Image Build

### Build Command

```bash
podman build -t ubi-vllm:latest -f containers/ubi-vllm/Containerfile containers/ubi-vllm/
```

**Alternative (Docker)**:
```bash
docker build -t ubi-vllm:latest -f containers/ubi-vllm/Containerfile containers/ubi-vllm/
```

### Build Arguments

| Argument | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `VLLM_VERSION` | String | No | `0.6.6` | vLLM package version to install |
| `PYTHON_VERSION` | String | No | `3.11` | Python version (3.11 or 3.12) |
| `CUDA_VERSION` | String | No | `12.6.1` | CUDA runtime version |

**Example with build args**:
```bash
podman build \
  --build-arg VLLM_VERSION=0.6.6 \
  --build-arg PYTHON_VERSION=3.11 \
  -t ubi-vllm:0.6.6 \
  -f containers/ubi-vllm/Containerfile \
  containers/ubi-vllm/
```

### Build Outputs

| Output | Type | Description |
|--------|------|-------------|
| Exit Code | Integer | `0` on success, non-zero on failure |
| Image ID | String | SHA256 image identifier |
| Image Tag | String | Tag applied to built image |
| Build Logs | stdout | Layer-by-layer build progress |

### Build Validation

Post-build validation commands:

```bash
# Verify image exists
podman images | grep ubi-vllm

# Check image size
podman inspect ubi-vllm:latest --format '{{.Size}}'

# Test vLLM import
podman run --rm ubi-vllm:latest python -c "import vllm; print(vllm.__version__)"

# Verify CUDA libraries
podman run --rm ubi-vllm:latest ls /usr/local/cuda/lib64/libcudart.so
```

---

## Container Instance Management

### Run Command (Basic)

```bash
podman run --device nvidia.com/gpu=all \
  -v ~/.cache/huggingface:/root/.cache/huggingface:Z \
  -e HF_HOME=/root/.cache/huggingface \
  -p 8000:8000 \
  --ipc=host \
  --shm-size=16G \
  --name vllm-server \
  ubi-vllm:latest \
  --model meta-llama/Llama-2-7b-hf \
  --dtype auto
```

**Docker equivalent**:
```bash
docker run --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -e HF_HOME=/root/.cache/huggingface \
  -p 8000:8000 \
  --ipc=host \
  --shm-size=16G \
  --name vllm-server \
  ubi-vllm:latest \
  --model meta-llama/Llama-2-7b-hf \
  --dtype auto
```

### Runtime Flags

#### GPU Allocation

| Flag | Podman | Docker | Description |
|------|--------|--------|-------------|
| GPU Device | `--device nvidia.com/gpu=all` | `--gpus all` | Allocate all GPUs |
| Specific GPU | `--device nvidia.com/gpu=0` | `--gpus '"device=0"'` | Allocate GPU 0 only |
| Multi-GPU | `--device nvidia.com/gpu=0,1` | `--gpus '"device=0,1"'` | Allocate GPU 0 and 1 |

#### Volume Mounts

| Volume | Host Path | Container Path | Flags | Purpose |
|--------|-----------|----------------|-------|---------|
| HuggingFace Cache | `~/.cache/huggingface` | `/root/.cache/huggingface` | `:Z` | Model files and cache |
| Custom Models | `/opt/models` | `/models` | `:ro,Z` | Pre-downloaded models (read-only) |
| Logs | `/var/log/vllm` | `/app/logs` | `:Z` | Persistent application logs |
| Config | `/etc/vllm/config.json` | `/config/config.json` | `:ro,Z` | Custom configuration file |

**SELinux Labels**:
- `:Z` = Private label (recommended for single container)
- `:z` = Shared label (for multi-container access)
- Omit on non-SELinux systems (not recommended for RHEL)

#### Environment Variables

| Variable | Required | Example Value | Description |
|----------|----------|---------------|-------------|
| `HF_HOME` | Yes | `/root/.cache/huggingface` | HuggingFace cache directory |
| `HUGGING_FACE_HUB_TOKEN` | Conditional | `hf_xxxxx` | Required for gated models |
| `CUDA_VISIBLE_DEVICES` | No | `all` or `0,1` | GPU device visibility (default: all) |
| `VLLM_CACHE_ROOT` | No | `/cache/vllm` | vLLM cache directory |
| `VLLM_LOGGING_CONFIG_PATH` | No | `/config/logging.json` | Custom logging configuration |

#### Network Configuration

| Flag | Value | Description |
|------|-------|-------------|
| `-p` / `--publish` | `8000:8000` | Map container port 8000 to host port 8000 |
| `-p` | `8080:8000` | Map container port 8000 to host port 8080 |
| `--network` | `host` | Use host network (not recommended for production) |

#### IPC and Shared Memory

| Flag | Value | Description |
|------|-------|-------------|
| `--ipc` | `host` | **Required** for tensor parallelism |
| `--shm-size` | `16G` | Shared memory size (minimum 16GB) |

#### Resource Limits

| Flag | Example | Description |
|------|---------|-------------|
| `--memory` | `32G` | Maximum memory limit |
| `--cpus` | `8.0` | CPU limit (number of CPUs) |
| `--memory-swap` | `48G` | Memory + swap limit |

### vLLM Server Arguments

Arguments passed to the vLLM entrypoint (after image name):

| Argument | Type | Required | Example | Description |
|----------|------|----------|---------|-------------|
| `--model` | String | Yes | `meta-llama/Llama-2-7b-hf` | HuggingFace model ID or path |
| `--dtype` | Enum | No | `auto` | Data type: auto, float16, bfloat16, float32 |
| `--host` | String | No | `0.0.0.0` | Bind address (default: 0.0.0.0) |
| `--port` | Integer | No | `8000` | API server port (default: 8000) |
| `--gpu-memory-utilization` | Float | No | `0.9` | Fraction of GPU memory to use (0.0-1.0) |
| `--max-model-len` | Integer | No | `4096` | Maximum sequence length |
| `--tensor-parallel-size` | Integer | No | `1` | Number of GPUs for tensor parallelism |
| `--max-num-seqs` | Integer | No | `256` | Max concurrent sequences |
| `--api-key` | String | No | `token-abc123` | API authentication token |
| `--served-model-name` | String | No | `my-model` | Alias for model in API responses |

**Example with full arguments**:
```bash
podman run --device nvidia.com/gpu=all \
  -v ~/.cache/huggingface:/root/.cache/huggingface:Z \
  -e HF_HOME=/root/.cache/huggingface \
  -p 8000:8000 \
  --ipc=host \
  --shm-size=16G \
  --name vllm-server \
  ubi-vllm:latest \
  --model meta-llama/Llama-2-7b-hf \
  --dtype auto \
  --gpu-memory-utilization 0.9 \
  --max-model-len 4096 \
  --api-key my-secret-token
```

---

## Container Lifecycle Commands

### Start Container

```bash
# Start a stopped container
podman start vllm-server

# Start with logs attached
podman start -a vllm-server
```

### Stop Container

```bash
# Graceful shutdown (10 second timeout)
podman stop vllm-server

# Immediate shutdown
podman stop -t 0 vllm-server

# Force kill
podman kill vllm-server
```

### Restart Container

```bash
# Restart with graceful shutdown
podman restart vllm-server

# Restart with immediate shutdown
podman restart -t 0 vllm-server
```

### Remove Container

```bash
# Remove stopped container
podman rm vllm-server

# Force remove running container
podman rm -f vllm-server
```

### Container Status

```bash
# List running containers
podman ps

# List all containers (including stopped)
podman ps -a

# Inspect container details
podman inspect vllm-server

# View container logs
podman logs vllm-server

# Follow logs (tail -f style)
podman logs -f vllm-server

# View resource usage
podman stats vllm-server
```

---

## Health Check Commands

### HTTP Health Endpoint

```bash
# Basic health check
curl http://localhost:8000/health

# With timeout
curl --max-time 5 http://localhost:8000/health

# Health check with exit code
curl -f http://localhost:8000/health && echo "Healthy" || echo "Unhealthy"
```

**Expected Response**:
- **Status Code**: 200 OK
- **Body**: Plain text "OK" or empty

### Container Health Status

```bash
# Check if container is running
podman ps --filter name=vllm-server --format "{{.Status}}"

# Check container health (if healthcheck configured)
podman inspect vllm-server --format '{{.State.Health.Status}}'
```

### API Availability Test

```bash
# Test model listing endpoint
curl http://localhost:8000/v1/models

# Test simple inference
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-2-7b-hf",
    "prompt": "Hello",
    "max_tokens": 5
  }'
```

---

## Debug and Troubleshooting Commands

### Interactive Shell Access

```bash
# Open shell in running container
podman exec -it vllm-server /bin/bash

# Run command in container
podman exec vllm-server nvidia-smi

# Check Python packages
podman exec vllm-server pip list | grep vllm
```

### Log Inspection

```bash
# View last 100 lines
podman logs --tail 100 vllm-server

# View logs since timestamp
podman logs --since 2025-11-07T10:00:00 vllm-server

# Export logs to file
podman logs vllm-server > vllm-server.log 2>&1
```

### GPU Verification

```bash
# Check GPU visibility in container
podman exec vllm-server nvidia-smi

# Verify CUDA availability
podman exec vllm-server python -c "import torch; print(torch.cuda.is_available())"

# Check GPU memory usage
podman exec vllm-server nvidia-smi --query-gpu=memory.used,memory.total --format=csv
```

### Resource Monitoring

```bash
# Real-time resource usage
podman stats vllm-server

# One-time resource snapshot
podman stats --no-stream vllm-server

# Detailed container inspect
podman inspect vllm-server | jq '.[0].State'
```

---

## Container Export and Distribution

### Save Image to Archive

```bash
# Save image to tar file
podman save -o ubi-vllm-latest.tar ubi-vllm:latest

# Compress while saving
podman save ubi-vllm:latest | gzip > ubi-vllm-latest.tar.gz
```

### Load Image from Archive

```bash
# Load from tar file
podman load -i ubi-vllm-latest.tar

# Load from compressed file
gunzip -c ubi-vllm-latest.tar.gz | podman load
```

### Push to Registry

```bash
# Tag for registry
podman tag ubi-vllm:latest registry.example.com/ubi-vllm:latest

# Login to registry
podman login registry.example.com

# Push to registry
podman push registry.example.com/ubi-vllm:latest
```

### Pull from Registry

```bash
# Pull image
podman pull registry.example.com/ubi-vllm:latest

# Pull and run
podman run registry.example.com/ubi-vllm:latest
```

---

## Exit Codes and Error Handling

### Build Exit Codes

| Code | Meaning | Common Causes |
|------|---------|---------------|
| 0 | Success | Build completed successfully |
| 1 | Build error | Syntax error in Containerfile, package installation failure |
| 2 | Invalid usage | Incorrect podman/docker command syntax |
| 125 | Container runtime error | Permission issues, storage driver problems |

### Run Exit Codes

| Code | Meaning | Common Causes |
|------|---------|---------------|
| 0 | Clean exit | Container stopped normally |
| 1 | Application error | vLLM startup failure, model loading error |
| 125 | Runtime error | GPU not available, volume mount failure |
| 126 | Command not executable | Entrypoint not found or not executable |
| 127 | Command not found | Entrypoint command doesn't exist |
| 137 | SIGKILL (OOM) | Out of memory, killed by system |
| 139 | SIGSEGV | Segmentation fault (GPU driver issue, CUDA error) |

### Common Error Messages

| Error | Cause | Resolution |
|-------|-------|------------|
| `Error: GPU not found` | NVIDIA runtime not available | Install nvidia-container-toolkit |
| `Error: cannot access local variable` | Old vLLM version issue | Use vLLM 0.6.6+ |
| `CUDA out of memory` | Model too large for GPU | Reduce `--gpu-memory-utilization` or use smaller model |
| `SELinux denying access to /path` | Volume mount permission issue | Add `:Z` or `:z` to volume mount |
| `port already in use` | Port 8000 occupied | Use different port: `-p 8080:8000` |

---

## Example Workflows

### Development Workflow

```bash
# 1. Build image
podman build -t ubi-vllm:dev -f containers/ubi-vllm/Containerfile containers/ubi-vllm/

# 2. Run with test model
podman run --device nvidia.com/gpu=all \
  -v ~/.cache/huggingface:/root/.cache/huggingface:Z \
  -e HF_HOME=/root/.cache/huggingface \
  -p 8000:8000 \
  --ipc=host \
  --shm-size=16G \
  --name vllm-dev \
  ubi-vllm:dev \
  --model meta-llama/Llama-2-7b-hf \
  --dtype auto

# 3. Test inference
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "meta-llama/Llama-2-7b-hf", "prompt": "Test", "max_tokens": 10}'

# 4. Stop and remove
podman stop vllm-dev && podman rm vllm-dev
```

### Production Workflow

```bash
# 1. Pull from registry
podman pull registry.internal.example.com/ubi-vllm:1.0.0

# 2. Run with production config
podman run -d \
  --device nvidia.com/gpu=all \
  -v /opt/models:/models:ro,Z \
  -v /opt/vllm/cache:/root/.cache/huggingface:Z \
  -v /var/log/vllm:/app/logs:Z \
  -e HF_HOME=/root/.cache/huggingface \
  -p 8000:8000 \
  --ipc=host \
  --shm-size=32G \
  --memory=64G \
  --restart=on-failure \
  --name vllm-prod \
  registry.internal.example.com/ubi-vllm:1.0.0 \
  --model /models/llama-2-70b \
  --dtype bfloat16 \
  --tensor-parallel-size 4 \
  --api-key "${VLLM_API_KEY}"

# 3. Verify health
curl http://localhost:8000/health

# 4. Monitor
podman logs -f vllm-prod
```

---

## Contract Guarantees

### Podman Compatibility

- All commands tested with **Podman 4.0+** on RHEL 10
- Rootless mode supported (recommended for security)
- SELinux integration required (`:Z` or `:z` labels)

### Docker Compatibility

- All commands compatible with **Docker 20.0+**
- NVIDIA Docker runtime required (`--gpus` flag)
- SELinux labels optional (non-RHEL systems)

### Platform Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| Host OS | RHEL 10 | RHEL 10.0+ |
| NVIDIA Driver | 535+ | Latest stable |
| CUDA Runtime | 12.1+ | 12.6+ |
| Container Runtime | Podman 4.0 / Docker 20.0 | Latest stable |
| GPU Memory | 8 GB | 24 GB+ |
| System RAM | 16 GB | 32 GB+ |
| Storage | 50 GB free | 100 GB+ |
