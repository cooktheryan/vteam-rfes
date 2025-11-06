# Tasks: Podman Grafana with Persistent Storage

**Input**: Design documents from `/specs/001-podman-grafana-pvc/`
**Prerequisites**: plan.md (complete), spec.md (complete), research.md (complete), data-model.md (complete), contracts/deployment-contract.md (complete), quickstart.md (complete)

**Tests**: No automated tests requested. This is a documentation deliverable with manual validation procedures.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`
- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions
- Documentation deliverable - all work in `specs/001-podman-grafana-pvc/`
- No source code directory structure needed

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Documentation structure validation and prerequisites

- [ ] T001 Verify all design documents are complete in specs/001-podman-grafana-pvc/ (plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md)
- [ ] T002 [P] Create comprehensive deployment documentation in specs/001-podman-grafana-pvc/DEPLOYMENT.md
- [ ] T003 [P] Create troubleshooting guide in specs/001-podman-grafana-pvc/TROUBLESHOOTING.md

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core documentation sections that MUST be complete before ANY user story validation can begin

**⚠️ CRITICAL**: No user story validation can begin until this phase is complete

- [ ] T004 Document prerequisite validation commands in specs/001-podman-grafana-pvc/DEPLOYMENT.md (Podman version check, SELinux status, port availability, disk space)
- [ ] T005 Document SELinux configuration requirements in specs/001-podman-grafana-pvc/DEPLOYMENT.md (`:Z` flag explanation, permission handling)
- [ ] T006 [P] Document persistent volume directory creation procedure in specs/001-podman-grafana-pvc/DEPLOYMENT.md
- [ ] T007 [P] Document container image selection and version pinning in specs/001-podman-grafana-pvc/DEPLOYMENT.md

**Checkpoint**: Foundation documentation ready - user story validation can now begin in parallel

---

## Phase 3: User Story 1 - Deploy Grafana with Persistent Storage (Priority: P1) 🎯 MVP

**Goal**: Administrator can deploy Grafana with persistent storage following documented shell commands

**Independent Test**: Deploy Grafana on fresh system, create dashboard, verify accessibility at http://localhost:3000

### Implementation for User Story 1

- [ ] T008 [P] [US1] Document core deployment command with all required flags in specs/001-podman-grafana-pvc/DEPLOYMENT.md (podman run with volume mount and port mapping)
- [ ] T009 [P] [US1] Document container naming convention and management in specs/001-podman-grafana-pvc/DEPLOYMENT.md
- [ ] T010 [US1] Document initial Grafana access procedure in specs/001-podman-grafana-pvc/DEPLOYMENT.md (URL, default credentials, first login)
- [ ] T011 [US1] Document basic container status verification commands in specs/001-podman-grafana-pvc/DEPLOYMENT.md (podman ps, podman logs)
- [ ] T012 [US1] Create validation script in specs/001-podman-grafana-pvc/validate-deployment.sh to verify US1 acceptance criteria
- [ ] T013 [US1] Test deployment on clean RHEL/CentOS system following only DEPLOYMENT.md
- [ ] T014 [US1] Document common deployment errors in specs/001-podman-grafana-pvc/TROUBLESHOOTING.md (SELinux denials, port conflicts, image pull failures)

**Checkpoint**: At this point, User Story 1 should be fully functional - administrator can deploy Grafana with working persistent storage

---

## Phase 4: User Story 2 - Verify Data Persistence (Priority: P2)

**Goal**: Administrator can verify persistent storage is correctly configured and data survives container lifecycle events

**Independent Test**: Create dashboard, stop/remove container, redeploy, verify dashboard persists

### Implementation for User Story 2

- [ ] T015 [P] [US2] Document volume mount verification procedure in specs/001-podman-grafana-pvc/DEPLOYMENT.md (podman inspect, checking host directory)
- [ ] T016 [P] [US2] Document file system inspection commands in specs/001-podman-grafana-pvc/DEPLOYMENT.md (checking grafana.db and data files on host)
- [ ] T017 [US2] Document container lifecycle test procedure in specs/001-podman-grafana-pvc/DEPLOYMENT.md (stop, remove, recreate with same volume)
- [ ] T018 [US2] Document data persistence verification steps in specs/001-podman-grafana-pvc/DEPLOYMENT.md (create test dashboard, verify after recreation)
- [ ] T019 [US2] Create validation script in specs/001-podman-grafana-pvc/validate-persistence.sh to automate US2 acceptance criteria
- [ ] T020 [US2] Test persistence verification procedure following DEPLOYMENT.md
- [ ] T021 [US2] Document persistence troubleshooting in specs/001-podman-grafana-pvc/TROUBLESHOOTING.md (missing :Z flag, wrong mount path, permission issues)

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently - deployment works AND persistence is verified

---

## Phase 5: User Story 3 - Reproduce Deployment from Documentation (Priority: P3)

**Goal**: New team member can deploy Grafana on fresh system using only provided documentation without assistance

**Independent Test**: Provide documentation to unfamiliar user, verify successful deployment without help

### Implementation for User Story 3

- [ ] T022 [P] [US3] Document systemd service generation in specs/001-podman-grafana-pvc/DEPLOYMENT.md (podman generate systemd command with --new flag)
- [ ] T023 [P] [US3] Document systemd service installation procedure in specs/001-podman-grafana-pvc/DEPLOYMENT.md (user service directory, daemon-reload, enable, start)
- [ ] T024 [US3] Document user lingering configuration in specs/001-podman-grafana-pvc/DEPLOYMENT.md (loginctl enable-linger explanation and commands)
- [ ] T025 [US3] Document boot persistence verification in specs/001-podman-grafana-pvc/DEPLOYMENT.md (reboot test, service status check)
- [ ] T026 [P] [US3] Document maintenance procedures in specs/001-podman-grafana-pvc/MAINTENANCE.md (backup, restore, upgrade Grafana version)
- [ ] T027 [P] [US3] Document log access methods in specs/001-podman-grafana-pvc/MAINTENANCE.md (podman logs, journalctl for systemd service)
- [ ] T028 [US3] Create complete end-to-end validation script in specs/001-podman-grafana-pvc/validate-e2e.sh covering all US3 scenarios
- [ ] T029 [US3] Conduct peer review of documentation with team member unfamiliar with setup
- [ ] T030 [US3] Test complete deployment from scratch following only documentation (no prior knowledge)
- [ ] T031 [US3] Document systemd service troubleshooting in specs/001-podman-grafana-pvc/TROUBLESHOOTING.md (lingering not enabled, service file errors, boot failures)

**Checkpoint**: All user stories should now be independently functional - complete end-to-end deployment, verification, and reproducibility

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories and final validation

- [ ] T032 [P] Create comprehensive README.md in specs/001-podman-grafana-pvc/ linking to all documentation files
- [ ] T033 [P] Add architecture diagram to specs/001-podman-grafana-pvc/DEPLOYMENT.md showing container, volume, and host relationships
- [ ] T034 Review and consolidate common commands into reference section in specs/001-podman-grafana-pvc/DEPLOYMENT.md
- [ ] T035 [P] Document edge cases from spec.md in specs/001-podman-grafana-pvc/TROUBLESHOOTING.md (incorrect permissions, service not running, port conflicts, disk space, upgrades, SELinux blocking)
- [ ] T036 [P] Document security considerations in specs/001-podman-grafana-pvc/SECURITY.md (rootless vs rootful, credential management, network exposure, HTTPS recommendation)
- [ ] T037 Validate all shell commands are copy-paste ready with no placeholders
- [ ] T038 Verify deployment time meets success criteria (under 10 minutes) through timed test
- [ ] T039 Run complete quickstart.md validation following documented procedures
- [ ] T040 Update CLAUDE.md at repository root with final technology stack and commands

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational phase completion
  - User stories can then proceed in parallel (if multiple documenters available)
  - Or sequentially in priority order (P1 → P2 → P3)
- **Polish (Phase 6)**: Depends on all user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P2)**: Depends on User Story 1 deployment documentation - Builds on deployment to add persistence verification
- **User Story 3 (P3)**: Depends on User Stories 1 and 2 - Adds systemd automation and reproducibility on top of deployment and verification

### Within Each User Story

- Documentation before validation scripts
- Validation scripts before testing
- Testing before troubleshooting documentation
- Each story must be independently testable after completion

### Parallel Opportunities

- All Setup tasks marked [P] can run in parallel (T002, T003)
- All Foundational tasks marked [P] can run in parallel (T006, T007)
- Within User Story 1: T008, T009 can run in parallel; then T010-T011 can run in parallel
- Within User Story 2: T015, T016 can run in parallel before T017
- Within User Story 3: T022, T023 can run in parallel; T026, T027 can run in parallel
- Within Polish: T032, T033, T035, T036 can run in parallel

---

## Parallel Example: User Story 1

```bash
# Launch documentation tasks in parallel:
Task 1: "Document core deployment command with all required flags in specs/001-podman-grafana-pvc/DEPLOYMENT.md"
Task 2: "Document container naming convention and management in specs/001-podman-grafana-pvc/DEPLOYMENT.md"

# Then launch verification tasks in parallel:
Task 3: "Document initial Grafana access procedure in specs/001-podman-grafana-pvc/DEPLOYMENT.md"
Task 4: "Document basic container status verification commands in specs/001-podman-grafana-pvc/DEPLOYMENT.md"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Test deployment following only US1 documentation
5. Demo basic Grafana deployment with persistence

### Incremental Delivery

1. Complete Setup + Foundational → Documentation foundation ready
2. Add User Story 1 → Test independently → Basic deployment works (MVP!)
3. Add User Story 2 → Test independently → Persistence verified
4. Add User Story 3 → Test independently → Full reproducibility with auto-start
5. Each story adds value without breaking previous stories

### Parallel Team Strategy

With multiple technical writers/validators:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Writer A: User Story 1 (deployment documentation)
   - Writer B: User Story 2 (persistence verification) - can start partial work
   - Writer C: Troubleshooting guide (supporting all stories)
3. Sequential dependencies require US1 completion before US2 validation, US2 before US3

---

## Validation Criteria

### User Story 1 Validation (MVP)
- [ ] Administrator can deploy Grafana in under 10 minutes following DEPLOYMENT.md
- [ ] Grafana accessible at http://localhost:3000 within 60 seconds
- [ ] Dashboard created in Grafana persists to ~/grafana-data directory
- [ ] All commands are copy-paste ready with no placeholders

### User Story 2 Validation
- [ ] Administrator can verify volume mount with documented commands
- [ ] Data files visible in host directory
- [ ] Dashboard survives container stop/remove/recreate cycle
- [ ] 100% of configuration data persists

### User Story 3 Validation
- [ ] New team member successfully deploys without assistance
- [ ] Systemd service configured and enabled
- [ ] Container survives system reboot
- [ ] All prerequisites, deployment, verification, troubleshooting documented

---

## Notes

- [P] tasks = different files/sections, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- All shell commands must be tested on actual RHEL/CentOS system
- Validation scripts should check acceptance criteria automatically where possible
- Documentation must work with both rootless and rootful Podman
- SELinux enforcing mode is the target environment
- This is a documentation deliverable - no application source code to write
