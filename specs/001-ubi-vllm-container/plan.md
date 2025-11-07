# Implementation Plan: UBI-based vLLM Container for RHEL 10

**Branch**: `001-ubi-vllm-container` | **Date**: 2025-11-07 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-ubi-vllm-container/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Create a containerized deployment of the vLLM inference engine using Red Hat Universal Base Image (UBI) for RHEL 10 systems. The solution includes a Dockerfile/Containerfile definition and comprehensive documentation covering build, deployment, configuration, and lifecycle management commands. The container must support GPU acceleration, configurable model loading, and work with both Podman and Docker runtimes.

## Technical Context

**Language/Version**: Python 3.11+ (vLLM requirement), Bash for deployment scripts
**Primary Dependencies**: vLLM (NEEDS CLARIFICATION - specific version), Red Hat UBI (NEEDS CLARIFICATION - which UBI variant: ubi9-python or ubi9-minimal), CUDA runtime (NEEDS CLARIFICATION - version compatible with vLLM and RHEL 10)
**Storage**: Container images (local registry or remote), mounted volumes for model files, persistent storage for logs (NEEDS CLARIFICATION - recommended volume mount strategy)
**Testing**: Container smoke tests (build verification, runtime startup, inference endpoint validation), NEEDS CLARIFICATION - testing framework/approach
**Target Platform**: RHEL 10 with Podman 4.x+ or Docker 20.x+, NVIDIA GPU support (CUDA-capable)
**Project Type**: Container infrastructure - single deliverable (Containerfile + documentation)
**Performance Goals**: Container startup <5 minutes for typical models, inference latency depends on model/GPU (out of scope for container optimization), NEEDS CLARIFICATION - memory footprint targets
**Constraints**: Must use only UBI base images (enterprise support requirement), GPU memory limits model size, container size should be minimized for distribution (NEEDS CLARIFICATION - target container image size)
**Scale/Scope**: Single-node deployment, designed for admin/operator use (not multi-tenant), documentation must cover 12+ operational commands (build, run, configure, lifecycle)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

**Initial Status (Pre-Phase 0)**: PASS - No project constitution defined yet. This is a greenfield infrastructure project with no existing architectural constraints to validate against.

**Post-Phase 1 Re-evaluation**: PASS - Design artifacts completed successfully.

**Design Validation**:
- ✅ **Architecture**: Container infrastructure pattern appropriate for deployment automation
- ✅ **Documentation**: Comprehensive documentation generated (research.md, data-model.md, quickstart.md, contracts)
- ✅ **Technology Choices**: All technical decisions documented with rationale in research.md
- ✅ **Testing Strategy**: Multi-layered testing approach defined (structure tests, smoke tests, integration tests, GPU tests)
- ✅ **Security**: SELinux integration, rootless Podman support, minimal attack surface considerations included
- ✅ **Completeness**: All Phase 0 and Phase 1 deliverables generated

**Note**: Once a constitution is established, this section should verify:
- Compliance with container image standards and policies
- Alignment with enterprise deployment practices
- Security and vulnerability scanning requirements
- Documentation and testing standards

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
containers/
└── ubi-vllm/
    ├── Containerfile           # UBI-based container definition for vLLM
    ├── entrypoint.sh           # Container entrypoint script
    ├── config/
    │   └── vllm-default.json   # Default vLLM configuration
    └── README.md               # Build and deployment documentation

docs/
└── ubi-vllm/
    ├── quickstart.md           # Quick start guide (Phase 1 output)
    ├── configuration.md        # Configuration reference
    ├── troubleshooting.md      # Common issues and solutions
    └── examples/
        ├── basic-deployment.sh
        ├── gpu-configuration.sh
        └── multi-model.sh

tests/
└── containers/
    └── ubi-vllm/
        ├── smoke-test.sh       # Basic container functionality tests
        ├── integration/
        │   ├── test-build.sh
        │   ├── test-startup.sh
        │   └── test-inference.sh
        └── fixtures/
            └── test-model/      # Small test model for validation
```

**Structure Decision**: Container infrastructure project with focus on deployment artifacts and comprehensive documentation. The `containers/ubi-vllm/` directory contains the core Containerfile and runtime configuration, while `docs/ubi-vllm/` provides user-facing operational guides. Testing is organized under `tests/containers/` to support future container additions.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
