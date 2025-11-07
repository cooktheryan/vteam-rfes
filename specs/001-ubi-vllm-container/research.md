# Research: UBI-based vLLM Container for RHEL 10

**Feature**: 001-ubi-vllm-container
**Date**: 2025-11-07
**Purpose**: Resolve technical unknowns identified in Technical Context

## Decision Summary

| Topic | Decision | Rationale |
|-------|----------|-----------|
| vLLM Version | 0.6.6 | Stable release before V1 refactoring, proven production track record |
| UBI Base Image | `nvcr.io/nvidia/cuda:12.6.1-runtime-ubi9` | CUDA pre-installed, smaller than devel, GPU-optimized |
| Python Version | Python 3.11 | vLLM requirement, available in UBI9 repos |
| CUDA Runtime | CUDA 12.1/12.6 | Compatible with vLLM 0.6.6 and RHEL 10 drivers |
| Volume Strategy | Runtime mount for models | Flexibility, smaller images, standard practice |
| Testing Framework | pytest + container-structure-test + BATS | Comprehensive coverage, GPU/non-GPU testing |
| Container Size Target | 8-10 GB | Realistic with multi-stage builds, excludes models |
| Memory Footprint | 20-100+ GB RAM | Variable by model size, 40-80 GB typical |

---

## 1. vLLM Version and Dependencies

### Decision: vLLM 0.6.6

**Key Dependencies**:
- **PyTorch**: 2.5.1
- **Python**: 3.11 or 3.12 (minimum 3.9)
- **CUDA**: 12.1 or 12.6
- **Transformers**: 4.46.3 - 4.48.1
- **xformers**: 0.0.28.post3
- **Ray**: >=2.9
- **NumPy**: <2.0.0 (constraint required)

**Rationale**:
1. **Stability**: Last stable release before major V1 architecture refactoring (0.7.0+)
2. **Production Proven**: Widely deployed through late 2024 and early 2025
3. **CUDA Compatibility**: Built with CUDA 12.1, forward compatible to 12.8
4. **PyTorch Ecosystem**: Uses stable PyTorch 2.5.1 with matching xformers
5. **RHEL 10 Compatible**: Works with UBI 10's glibc 2.39 and kernel 6.12

**Alternatives Considered**:
- **vLLM 0.7.0+**: REJECTED - V1 alpha architecture still experimental
- **vLLM 0.10.1.1+**: DEFERRED - Red Hat's version but lacks current UBI 10 deployment patterns
- **vLLM 0.6.3/0.6.4**: REJECTED - Lacks bug fixes and stability improvements

**UBI Compatibility Notes**:
- Use pre-built wheels via pip (avoid source compilation)
- Install `openssl-devel` if building from source
- Validated on RHEL 9/UBI 9 architecture
- RHEL 10 provides improved GPU support with kernel 6.12

---

## 2. UBI Base Image Selection

### Decision: `nvcr.io/nvidia/cuda:12.6.1-runtime-ubi9`

**Image Characteristics**:
- **Size**: ~1.5-2 GB compressed (~4-5 GB uncompressed)
- **Python**: NOT pre-installed (install via microdnf)
- **CUDA**: Full runtime libraries, cuBLAS, NVIDIA driver libs
- **Compilation**: NO development tools (keeps image smaller)

**Rationale**:
1. **GPU-First Approach**: Easier to add Python to CUDA base than CUDA to Python base
2. **Size Efficiency**: 40-50% smaller than devel variants
3. **Official Support**: NVIDIA-maintained, UBI9-based
4. **Runtime Optimized**: Pre-built vLLM wheels don't need compilation
5. **Enterprise Compatible**: UBI base provides Red Hat support

**Alternatives Considered**:
- **ubi9/python-311**: REJECTED - No CUDA support, complex to retrofit
- **ubi9-minimal**: REJECTED - Must install Python AND CUDA runtime
- **nvidia/cuda:*-devel-ubi9**: REJECTED - Larger (~3-4 GB), unnecessary compilers
- **Red Hat AI Inference Server**: Alternative for enterprise users with subscriptions

**Build Implications**:
```dockerfile
FROM nvcr.io/nvidia/cuda:12.6.1-runtime-ubi9
RUN microdnf install -y python3.11 python3.11-pip python3.11-devel && \
    microdnf clean all
RUN pip3.11 install vllm==0.6.6
```

**Final Image Size Estimate**: 8-10 GB with optimization

---

## 3. Container Volume Strategy

### Decision: Runtime Mount for Model Files

**Recommended Mount Points**:

| Mount Point | Host Path Example | Container Path | Purpose |
|-------------|-------------------|----------------|---------|
| HuggingFace Cache | `~/.cache/huggingface` | `/root/.cache/huggingface` | Model files, tokenizers |
| Shared Memory | `/dev/shm` | `/dev/shm` | IPC for tensor parallelism |
| Logs | `/var/log/vllm` | `/app/logs` | Persistent application logs |
| vLLM Cache | Custom | Set via `VLLM_CACHE_ROOT` | Compiled kernels, optimizations |

**Rationale**:
1. **Flexibility**: Update models without rebuilding containers
2. **Size Optimization**: LLM models are 10-100+ GB, would create massive images
3. **Standard Practice**: Aligns with official vLLM documentation
4. **Cost Efficiency**: Avoid duplicate model storage across images
5. **Operational**: Faster deployments and updates

**SELinux Configuration for RHEL**:
- Use `:Z` for exclusive container access (private label)
- Use `:z` for shared access across multiple containers
- Never disable SELinux in production

**Example Podman Command**:
```bash
podman run --device nvidia.com/gpu=all \
  -v ~/.cache/huggingface:/root/.cache/huggingface:Z \
  -v /var/log/vllm:/app/logs:Z \
  -e HF_HOME=/root/.cache/huggingface \
  -p 8000:8000 \
  --ipc=host \
  --shm-size=16G \
  ubi-vllm:latest
```

**Logging Strategy**:
- **Primary**: stdout/stderr (captured by container runtime)
- **Secondary**: Persistent volume for audit/compliance logs
- **Configuration**: Use `VLLM_LOGGING_CONFIG_PATH` for custom logging

---

## 4. Testing Framework and Strategy

### Decision: Hybrid Approach - pytest + container-structure-test + BATS

**Framework Roles**:
- **pytest with docker-py**: Primary framework for programmatic container testing
- **container-structure-test**: Declarative YAML-based image validation
- **BATS**: Shell script validation and operational workflow testing

**Essential Smoke Tests**:
1. **Build Validation**: Image builds successfully, correct size, no vulnerabilities
2. **Startup Validation**: Container starts, health endpoint available, no errors
3. **Inference Functionality**: API endpoints respond correctly
4. **Configuration Validation**: Environment variables applied, mounts accessible
5. **Runtime Stability**: No crashes, stable memory usage, graceful shutdown

**GPU Testing Strategy**:

**With GPU Hardware**:
```bash
# Verify GPU accessible
podman run --rm --device nvidia.com/gpu=all <image> nvidia-smi

# Check CUDA availability
python -c "import torch; print(torch.cuda.is_available())"
```

**Without GPU (CI/CD)**:
1. **Build-time tests**: Container structure, dependencies, configuration
2. **CPU-mode tests**: Start container with `CUDA_VISIBLE_DEVICES=""`, test API
3. **Conditional skipping**: Use pytest markers to skip GPU tests
4. **Separate pipelines**: Standard (CPU) vs GPU (self-hosted runner)

**Rationale**:
- **Flexibility**: pytest handles complex scenarios, container-structure-test provides fast validation
- **Pragmatic**: Layered approach works with/without GPU access
- **Cost-effective**: Most tests run on standard CI without expensive GPU runners
- **RHEL Compatible**: Works seamlessly with Podman and Buildah

**Test Structure**:
```
tests/containers/ubi-vllm/
├── structure/          # container-structure-test configs
├── smoke/              # pytest + BATS basic tests
├── integration/        # pytest complex scenarios
├── gpu/                # pytest GPU-specific tests
└── fixtures/           # Test models and data
```

---

## 5. Container Size Optimization

### Decision: Target 8-10 GB (excluding model files)

**Size Breakdown**:
- **UBI Python 3.11 base**: ~1 GB
- **CUDA runtime libraries**: ~2 GB
- **PyTorch with CUDA**: ~2.5-3 GB
- **vLLM + dependencies**: ~1.5-2 GB
- **Optimization overhead**: ~1-2 GB

**Memory Footprint During Operation**:
- **Small models**: 20-30 GB RAM
- **Typical deployments**: 40-80 GB RAM
- **Large models (70B+)**: 100+ GB RAM
- **GPU Memory**: Pre-allocates maximum available for KV cache

**Optimization Techniques**:
1. **Multi-stage builds** (ESSENTIAL): 40%+ size reduction
   - Build stage: Use devel image with compilers
   - Runtime stage: Copy only compiled artifacts
2. **GPU architecture targeting**: Build only for target GPU (30-40% reduction)
3. **Layer optimization**: Combine RUN commands, order by change frequency
4. **Exclude build dependencies**: Remove compilers, headers, static libs
5. **Clean package caches**: `pip install --no-cache-dir`, `microdnf clean all`

**Multi-stage Build Strategy**: STRONGLY RECOMMENDED

**Example Pattern**:
```dockerfile
# Stage 1: Build
FROM nvidia/cuda:12.6.1-devel-ubi9 AS builder
# Install build dependencies, compile vLLM

# Stage 2: Runtime
FROM nvidia/cuda:12.6.1-runtime-ubi9
# Copy only runtime artifacts from builder
```

**Rationale**:
- **Balance**: 8-10 GB provides full functionality while remaining distributable
- **Industry Standard**: Comparable to other production ML containers
- **Security**: Smaller attack surface without build tools
- **Trade-offs**: Prioritizes functionality over extreme minimization

**Baseline Comparison**:
- Official vLLM image: 12.6 GB (all GPU architectures)
- Optimized vLLM: 6.9 GB (specific GPU targeting)
- Typical ML containers: 4-8 GB (production-optimized)

---

## Implementation Notes

### Installation Method
```bash
# Use pre-built wheels to avoid compilation issues
pip install vllm==0.6.6 --no-cache-dir
```

### Required Environment Variables
```bash
HF_HOME=/root/.cache/huggingface
VLLM_CACHE_ROOT=/cache/vllm
CUDA_VISIBLE_DEVICES=all
```

### Host Requirements
- RHEL 10 with Podman 4.x+ or Docker 20.x+
- NVIDIA GPU drivers version 535+
- NVIDIA Container Toolkit
- 16GB+ system RAM (32GB+ recommended)
- GPU memory: 8GB minimum, 24GB+ for production

### Security Considerations
- Use SELinux labels (`:z` or `:Z`) on all volume mounts
- Run rootless Podman when possible
- Mount model directories read-only (`:ro`) when applicable
- Never use `--privileged` in production
- Scan images for vulnerabilities regularly

---

## References

- vLLM Documentation: https://docs.vllm.ai/en/v0.6.6/
- Red Hat UBI Catalog: https://catalog.redhat.com/
- NVIDIA CUDA Container Images: https://hub.docker.com/r/nvidia/cuda
- Red Hat AI Inference Server: https://www.redhat.com/en/products/ai/inference-server
- Container Testing Best Practices: Google container-structure-test, pytest documentation
