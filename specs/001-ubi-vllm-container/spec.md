# Feature Specification: UBI-based vLLM Container for RHEL 10

**Feature Branch**: `001-ubi-vllm-container`
**Created**: 2025-11-06
**Status**: Draft
**Input**: User description: "the container must use UBI to run vllm and then provide me all the commands required to run it on my rhel 10 system"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Container Build and Deployment (Priority: P1)

As a system administrator, I need to build and deploy a containerized vLLM service on my RHEL 10 system using a UBI-based container image, so that I can run large language model inference workloads in a supported, enterprise-grade environment.

**Why this priority**: This is the core MVP functionality - without a working container build and deployment process, the feature provides no value. This establishes the foundation for all other capabilities.

**Independent Test**: Can be fully tested by providing a container definition file, building the container, and successfully starting the vLLM service. Delivers immediate value by enabling users to run vLLM inference workloads.

**Acceptance Scenarios**:

1. **Given** a RHEL 10 system with container runtime installed, **When** a user follows the provided build instructions, **Then** a UBI-based container image containing vLLM is successfully created
2. **Given** a built vLLM container image, **When** a user executes the provided run commands, **Then** the vLLM service starts successfully and is accessible for inference requests
3. **Given** a running vLLM container, **When** a user sends a test inference request, **Then** the service responds with valid model output

---

### User Story 2 - Container Configuration and Customization (Priority: P2)

As a system administrator, I need clear documentation on how to configure the vLLM container with custom settings (model selection, GPU allocation, resource limits), so that I can optimize the service for my specific hardware and workload requirements.

**Why this priority**: While the basic container deployment (P1) provides core functionality, real-world usage requires customization for different models, hardware configurations, and resource constraints. This makes the solution production-ready.

**Independent Test**: Can be tested independently by modifying configuration parameters (model path, GPU settings, memory limits) and verifying the container starts with the updated settings and performs inference correctly.

**Acceptance Scenarios**:

1. **Given** documentation for container configuration, **When** a user specifies a custom model path, **Then** the container loads and serves the specified model
2. **Given** a multi-GPU RHEL 10 system, **When** a user configures GPU allocation parameters, **Then** vLLM utilizes the specified GPUs for inference
3. **Given** resource constraint requirements, **When** a user sets memory and CPU limits, **Then** the container operates within the defined resource boundaries

---

### User Story 3 - Container Lifecycle Management (Priority: P3)

As a system administrator, I need commands and documentation for managing the vLLM container lifecycle (start, stop, restart, update, health check), so that I can maintain and troubleshoot the service in production environments.

**Why this priority**: While deployment and configuration (P1, P2) enable initial usage, production deployments require ongoing operational management. This enhances operational maturity but isn't needed for initial MVP validation.

**Independent Test**: Can be tested by executing lifecycle commands (stop, start, restart, health check) and verifying expected container state transitions and service availability.

**Acceptance Scenarios**:

1. **Given** a running vLLM container, **When** a user executes the stop command, **Then** the container gracefully shuts down without data loss
2. **Given** a stopped vLLM container, **When** a user executes the start command, **Then** the container restarts and resumes serving inference requests
3. **Given** a running vLLM container, **When** a user executes a health check command, **Then** the system reports the service status and resource utilization

---

### Edge Cases

- What happens when the RHEL 10 system doesn't have GPU drivers installed or GPU hardware available?
- How does the container handle insufficient memory for loading large language models?
- What happens when the specified model path doesn't exist or contains incompatible model files?
- How does the system behave when the container runtime (Podman/Docker) is not installed or is an incompatible version?
- What happens when network connectivity is required for model downloads but is unavailable?
- How does the container handle concurrent inference requests that exceed available GPU memory?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Container image MUST be based on Red Hat Universal Base Image (UBI) to ensure enterprise support and compatibility
- **FR-002**: Container MUST include vLLM inference engine with all required dependencies pre-installed
- **FR-003**: Documentation MUST provide complete step-by-step commands for building the container image on RHEL 10
- **FR-004**: Documentation MUST provide complete step-by-step commands for running the container on RHEL 10
- **FR-005**: Container MUST support GPU acceleration for model inference when GPU hardware is available
- **FR-006**: Documentation MUST include commands for basic container lifecycle operations (start, stop, restart, status check)
- **FR-007**: Container MUST expose a network interface for accepting inference requests from external clients
- **FR-008**: Documentation MUST include example commands for testing the vLLM service with sample inference requests
- **FR-009**: Container configuration MUST allow users to specify which language model to load
- **FR-010**: Documentation MUST specify minimum system requirements (memory, GPU, storage) for running the container
- **FR-011**: Container MUST handle startup failures gracefully with clear error messages for common issues (missing GPU drivers, insufficient memory, missing models)
- **FR-012**: Documentation MUST work with both Podman and Docker container runtimes available on RHEL 10

### Non-Functional Requirements

- **NFR-001**: Container build process should complete within reasonable time on standard RHEL 10 hardware (guidance: under 30 minutes with good network connectivity)
- **NFR-002**: Documentation must be clear enough for system administrators with basic container knowledge to follow without external assistance
- **NFR-003**: Container startup time should be reasonable for production use (guidance: under 5 minutes for typical models)

### Key Entities

- **Container Image**: The built artifact containing UBI base, vLLM, dependencies, and configuration. Stored in local image registry or external container registry. Can be versioned and distributed.
- **Language Model**: The AI model files that vLLM loads for inference. Located in filesystem path accessible to container (mounted volume or embedded in image). Varies in size from gigabytes to hundreds of gigabytes.
- **Container Instance**: The running container process on RHEL 10 system. Has allocated resources (CPU, memory, GPU), network ports, and mounted volumes. Has lifecycle states (stopped, running, failed).
- **Documentation Artifact**: The complete set of commands and instructions. Includes build commands, run commands, configuration examples, troubleshooting steps, and system requirements.

### Assumptions

- RHEL 10 system has a supported container runtime (Podman or Docker) installed
- User has sufficient privileges to build and run containers on the RHEL 10 system
- System has internet connectivity for downloading base images and dependencies during build (or alternative mechanism for air-gapped environments is out of scope)
- vLLM version will be a recent stable release compatible with available UBI Python runtime
- GPU support assumes NVIDIA GPUs with appropriate drivers installed (AMD GPU support out of scope for initial version)
- Model files are user-provided and not included in the container image (mounted at runtime)

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user with basic container knowledge can successfully build the UBI-based vLLM container image by following the provided documentation without external assistance
- **SC-002**: A user can successfully deploy and start the vLLM container on a RHEL 10 system within 10 minutes of completing the build
- **SC-003**: The deployed vLLM service successfully processes inference requests and returns valid responses for standard language model tasks
- **SC-004**: The container starts successfully on RHEL 10 systems with both Podman and Docker runtimes
- **SC-005**: Documentation includes all necessary commands for the complete workflow from build through testing, requiring no external documentation lookup
- **SC-006**: Users can customize the container configuration (model selection, resource limits) by modifying clearly documented parameters
- **SC-007**: 90% of users can complete the entire workflow (build, deploy, test) on their first attempt without encountering blocking errors

### Definition of Done

This feature is complete when:
1. A container definition file (Dockerfile/Containerfile) based on UBI exists and successfully builds a vLLM-capable image
2. Complete documentation exists with step-by-step commands for building the container on RHEL 10
3. Complete documentation exists with step-by-step commands for running the container on RHEL 10
4. Documentation includes example commands for testing vLLM inference functionality
5. Documentation includes configuration examples for common customization scenarios (model selection, GPU allocation, resource limits)
6. Documentation includes troubleshooting guidance for common failure scenarios
7. The solution has been validated on a RHEL 10 system with both Podman and Docker
8. Documentation specifies minimum system requirements clearly
