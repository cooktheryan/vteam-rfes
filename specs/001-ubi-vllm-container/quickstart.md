# Quick Start: UBI vLLM Container on RHEL 10

**Get started with vLLM inference in under 10 minutes**

---

## Prerequisites

Before you begin, ensure you have:

- ✅ RHEL 10 system with root or sudo access
- ✅ NVIDIA GPU (8GB+ VRAM recommended)
- ✅ 50GB+ free disk space
- ✅ Internet connection for downloading models and dependencies

**Minimum System Requirements**:
- OS: RHEL 10.0+
- GPU: NVIDIA GPU with CUDA support (Compute Capability 7.0+)
- GPU Memory: 8GB minimum, 24GB+ recommended
- RAM: 16GB minimum, 32GB+ recommended
- Storage: 50GB+ (models can be 10-100+ GB each)

---

## Step 1: Install Prerequisites

### Install Container Runtime (Podman)

Podman is the default container engine for RHEL 10:

```bash
# Podman should be pre-installed on RHEL 10
# If not, install it:
sudo dnf install -y podman

# Verify installation
podman --version
# Expected: podman version 4.0.0 or higher
```

**Alternative: Docker**
```bash
# If you prefer Docker:
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
```

### Install NVIDIA GPU Drivers

```bash
# Check if drivers are already installed
nvidia-smi

# If not installed, follow NVIDIA's official RHEL 10 driver installation:
# https://docs.nvidia.com/datacenter/tesla/tesla-installation-notes/index.html

# Verify driver version (535+ required)
nvidia-smi --query-gpu=driver_version --format=csv,noheader
```

### Install NVIDIA Container Toolkit

```bash
# Add NVIDIA container toolkit repository
curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo \
  | sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo

# Install the toolkit
sudo dnf install -y nvidia-container-toolkit

# Configure Podman to use NVIDIA runtime
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

# Restart Podman (if running)
sudo systemctl restart podman || true
```

**Verification**:
```bash
# Test GPU access in container
podman run --rm --device nvidia.com/gpu=all nvidia/cuda:12.0.0-base-ubi9 nvidia-smi
```

---

## Step 2: Build the Container Image

### Clone or Download Containerfile

```bash
# Create project directory
mkdir -p ~/ubi-vllm-project
cd ~/ubi-vllm-project

# For this quickstart, we'll create a minimal Containerfile inline
# (In production, use the full Containerfile from the repository)
```

### Create Minimal Containerfile

```bash
cat > Containerfile << 'EOF'
FROM nvcr.io/nvidia/cuda:12.6.1-runtime-ubi9

# Install Python 3.11
RUN microdnf install -y python3.11 python3.11-pip && \
    microdnf clean all

# Create symlinks for python/pip
RUN ln -sf /usr/bin/python3.11 /usr/bin/python && \
    ln -sf /usr/bin/pip3.11 /usr/bin/pip

# Upgrade pip
RUN pip install --no-cache-dir --upgrade pip

# Install vLLM
RUN pip install --no-cache-dir vllm==0.6.6

# Set working directory
WORKDIR /app

# Expose vLLM API port
EXPOSE 8000

# Set environment variables
ENV HF_HOME=/root/.cache/huggingface
ENV VLLM_CACHE_ROOT=/cache/vllm

# Entrypoint
ENTRYPOINT ["python", "-m", "vllm.entrypoints.openai.api_server"]
EOF
```

### Build the Image

```bash
# Build with Podman (5-15 minutes depending on network speed)
podman build -t ubi-vllm:latest -f Containerfile .

# Verify the build
podman images | grep ubi-vllm

# Test vLLM installation
podman run --rm ubi-vllm:latest python -c "import vllm; print(f'vLLM version: {vllm.__version__}')"
```

**Expected output**:
```
REPOSITORY    TAG       IMAGE ID       CREATED          SIZE
ubi-vllm      latest    <image-id>     X minutes ago    8-10 GB
```

---

## Step 3: Run the Container

### Basic Deployment (Small Model)

For your first test, we'll use a small 7B parameter model:

```bash
# Create cache directory for models
mkdir -p ~/.cache/huggingface

# Run the container
podman run -d \
  --device nvidia.com/gpu=all \
  -v ~/.cache/huggingface:/root/.cache/huggingface:Z \
  -e HF_HOME=/root/.cache/huggingface \
  -p 8000:8000 \
  --ipc=host \
  --shm-size=16G \
  --name vllm-server \
  ubi-vllm:latest \
  --model meta-llama/Llama-2-7b-hf \
  --dtype auto \
  --max-model-len 2048
```

**Docker equivalent**:
```bash
docker run -d \
  --gpus all \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  -e HF_HOME=/root/.cache/huggingface \
  -p 8000:8000 \
  --ipc=host \
  --shm-size=16G \
  --name vllm-server \
  ubi-vllm:latest \
  --model meta-llama/Llama-2-7b-hf \
  --dtype auto \
  --max-model-len 2048
```

**What's happening**:
1. Container downloads Llama-2-7B model (~13GB) on first run
2. vLLM loads the model into GPU memory
3. API server starts on port 8000
4. This may take 3-5 minutes on first run

### Monitor Startup

```bash
# Follow the logs
podman logs -f vllm-server

# Watch for these key messages:
# - "Downloading model..." (first run only)
# - "Loading model..."
# - "Initialized engine"
# - "Uvicorn running on http://0.0.0.0:8000"
```

**Startup is complete when you see**:
```
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

Press `Ctrl+C` to stop following logs (container keeps running in background).

---

## Step 4: Test the Deployment

### Health Check

```bash
# Check if the server is ready
curl http://localhost:8000/health

# Expected output:
# OK
```

### List Available Models

```bash
curl http://localhost:8000/v1/models | jq

# Expected output:
# {
#   "object": "list",
#   "data": [
#     {
#       "id": "meta-llama/Llama-2-7b-hf",
#       "object": "model",
#       ...
#     }
#   ]
# }
```

### Generate Text Completion

```bash
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-2-7b-hf",
    "prompt": "The capital of France is",
    "max_tokens": 20,
    "temperature": 0.7
  }' | jq
```

**Expected output** (example):
```json
{
  "id": "cmpl-abc123",
  "object": "text_completion",
  "created": 1699000000,
  "model": "meta-llama/Llama-2-7b-hf",
  "choices": [
    {
      "text": " Paris, and it is home to the Eiffel Tower, the Louvre Museum",
      "index": 0,
      "finish_reason": "length"
    }
  ],
  "usage": {
    "prompt_tokens": 5,
    "completion_tokens": 20,
    "total_tokens": 25
  }
}
```

### Chat Completion

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-2-7b-hf",
    "messages": [
      {"role": "user", "content": "What is containerization?"}
    ],
    "max_tokens": 100
  }' | jq
```

---

## Step 5: Container Management

### View Running Containers

```bash
# List running containers
podman ps

# Show resource usage
podman stats vllm-server
```

### Stop the Container

```bash
# Graceful shutdown
podman stop vllm-server

# Verify it's stopped
podman ps -a | grep vllm-server
```

### Start Again

```bash
# Start the stopped container
podman start vllm-server

# Follow logs
podman logs -f vllm-server
```

### Remove the Container

```bash
# Stop and remove
podman stop vllm-server
podman rm vllm-server

# Verify removal
podman ps -a | grep vllm-server
# (no output = successfully removed)
```

---

## Common Issues and Solutions

### Issue: "GPU not found"

**Symptoms**:
```
Error: GPU device not available
```

**Solution**:
```bash
# 1. Verify NVIDIA driver
nvidia-smi

# 2. Check NVIDIA container toolkit
nvidia-ctk cdi list

# 3. Regenerate CDI specification
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml

# 4. For Docker, verify nvidia-runtime
docker run --rm --gpus all nvidia/cuda:12.0.0-base-ubuntu22.04 nvidia-smi
```

---

### Issue: "CUDA out of memory"

**Symptoms**:
```
torch.cuda.OutOfMemoryError: CUDA out of memory
```

**Solution**:
```bash
# Option 1: Reduce GPU memory utilization
podman run ... ubi-vllm:latest \
  --model meta-llama/Llama-2-7b-hf \
  --gpu-memory-utilization 0.8  # Default is 0.9

# Option 2: Reduce context length
podman run ... ubi-vllm:latest \
  --model meta-llama/Llama-2-7b-hf \
  --max-model-len 2048  # Smaller than default

# Option 3: Use a smaller model
podman run ... ubi-vllm:latest \
  --model TinyLlama/TinyLlama-1.1B-Chat-v1.0
```

---

### Issue: "SELinux denying access"

**Symptoms**:
```
Error: cannot access /root/.cache/huggingface: Permission denied
```

**Solution**:
```bash
# Add SELinux label to volume mount (RHEL/Fedora)
# Change:
  -v ~/.cache/huggingface:/root/.cache/huggingface

# To:
  -v ~/.cache/huggingface:/root/.cache/huggingface:Z  # Note the :Z

# Or temporarily (NOT recommended for production):
sudo setenforce 0  # Permissive mode
```

---

### Issue: "Port already in use"

**Symptoms**:
```
Error: cannot bind to port 8000: address already in use
```

**Solution**:
```bash
# Option 1: Use a different host port
podman run ... -p 8080:8000 ... ubi-vllm:latest

# Then access via:
curl http://localhost:8080/health

# Option 2: Find and stop the conflicting service
sudo lsof -i :8000
sudo kill <PID>
```

---

### Issue: "Model download is slow/stalled"

**Solution**:
```bash
# Pre-download the model before running the container
pip install huggingface_hub
python -c "
from huggingface_hub import snapshot_download
snapshot_download('meta-llama/Llama-2-7b-hf', local_dir_use_symlinks=False)
"

# Then run container (model already cached)
podman run ... ubi-vllm:latest --model meta-llama/Llama-2-7b-hf
```

---

## Next Steps

Now that you have vLLM running, explore:

1. **Configuration Guide**: Learn about GPU settings, model parameters, and performance tuning
2. **Multi-Model Deployment**: Serve multiple models simultaneously
3. **Production Deployment**: Add monitoring, logging, and high availability
4. **Custom Models**: Load your own fine-tuned models
5. **API Integration**: Connect applications to the vLLM API

### Recommended Reading

- [vLLM Official Documentation](https://docs.vllm.ai/)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference) (compatible endpoint format)
- [RHEL Container Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/10/html/building_running_and_managing_containers/)
- [NVIDIA GPU Containers](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/overview.html)

### Sample Models to Try

| Model | Size | VRAM Required | Best For |
|-------|------|---------------|----------|
| `TinyLlama/TinyLlama-1.1B-Chat-v1.0` | 1.1B | ~4GB | Testing, development |
| `meta-llama/Llama-2-7b-hf` | 7B | ~14GB | General-purpose |
| `meta-llama/Llama-2-13b-hf` | 13B | ~26GB | Higher quality |
| `mistralai/Mistral-7B-v0.1` | 7B | ~14GB | Strong performance |
| `meta-llama/Llama-2-70b-hf` | 70B | ~140GB | Production, multi-GPU |

**Note**: VRAM requirements are approximate and depend on context length, batch size, and quantization.

---

## Complete Example Script

Save this as `quick-start.sh` for easy redeployment:

```bash
#!/bin/bash
set -e

echo "=== UBI vLLM Quick Start ==="

# Variables
CONTAINER_NAME="vllm-server"
MODEL="meta-llama/Llama-2-7b-hf"
PORT="8000"

# Stop and remove existing container if present
podman rm -f $CONTAINER_NAME 2>/dev/null || true

# Create cache directory
mkdir -p ~/.cache/huggingface

# Run container
echo "Starting vLLM server..."
podman run -d \
  --device nvidia.com/gpu=all \
  -v ~/.cache/huggingface:/root/.cache/huggingface:Z \
  -e HF_HOME=/root/.cache/huggingface \
  -p $PORT:8000 \
  --ipc=host \
  --shm-size=16G \
  --name $CONTAINER_NAME \
  ubi-vllm:latest \
  --model $MODEL \
  --dtype auto \
  --max-model-len 2048

# Wait for health endpoint
echo "Waiting for vLLM to be ready..."
for i in {1..60}; do
  if curl -s http://localhost:$PORT/health > /dev/null 2>&1; then
    echo "vLLM server is ready!"
    break
  fi
  echo -n "."
  sleep 5
done

# Test inference
echo -e "\nTesting inference..."
curl http://localhost:$PORT/v1/completions \
  -H "Content-Type: application/json" \
  -d "{\"model\": \"$MODEL\", \"prompt\": \"Hello, vLLM!\", \"max_tokens\": 20}" \
  | jq

echo -e "\n=== Quick Start Complete ==="
echo "Container: $CONTAINER_NAME"
echo "API Endpoint: http://localhost:$PORT"
echo "Health Check: curl http://localhost:$PORT/health"
echo "View Logs: podman logs -f $CONTAINER_NAME"
echo "Stop Container: podman stop $CONTAINER_NAME"
```

Run with:
```bash
chmod +x quick-start.sh
./quick-start.sh
```

---

**Congratulations!** You now have a fully functional vLLM inference server running in a UBI container on RHEL 10. 🎉
