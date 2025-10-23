# Tasks: MongoDB Operator Deployment

**Input**: Design documents from `/specs/001-mongodb-operator/`
**Prerequisites**: plan.md, spec.md, data-model.md, contracts/, research.md, quickstart.md

**Tests**: Based on research.md testing strategy - comprehensive testing with unit, integration, and E2E tests

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `- [ ] [ID] [P?] [Story] Description`
- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions
Per plan.md structure:
```
mongodb-operator/
├── api/v1alpha1/          # CRD definitions
├── controllers/           # Reconciliation logic
├── internal/              # Internal packages
├── config/                # Kubernetes manifests
└── tests/                 # Tests (unit, integration, e2e)
```

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and operator scaffolding

- [ ] T001 Initialize operator project with Operator SDK/Kubebuilder at repository root
- [ ] T002 [P] Configure Go modules and dependencies (controller-runtime, client-go, mongo-go-driver) in go.mod
- [ ] T003 [P] Setup Makefile targets (build, test, deploy, manifests) in Makefile
- [ ] T004 [P] Create .gitignore and project documentation structure
- [ ] T005 [P] Setup golangci-lint configuration in .golangci.yml

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core CRD definitions and operator framework that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### CRD Definitions

- [ ] T006 [P] Create MongoDBInstance CRD types in api/v1alpha1/mongodbinstance_types.go
- [ ] T007 [P] Create MongoDBBackup CRD types in api/v1alpha1/mongodbbackup_types.go
- [ ] T008 [P] Create MongoDBRestore CRD types in api/v1alpha1/mongodbrestore_types.go
- [ ] T009 [P] Implement validation webhooks for MongoDBInstance in api/v1alpha1/mongodbinstance_webhook.go
- [ ] T010 Generate CRD manifests using controller-gen in config/crd/bases/

### Core Operator Infrastructure

- [ ] T011 [P] Create controller reconciler scaffolding for MongoDBInstance in controllers/mongodbinstance_controller.go
- [ ] T012 [P] Create controller reconciler scaffolding for MongoDBBackup in controllers/mongodbbackup_controller.go
- [ ] T013 [P] Create controller reconciler scaffolding for MongoDBRestore in controllers/mongodbrestore_controller.go
- [ ] T014 Setup manager with all controllers registered in main.go
- [ ] T015 [P] Configure RBAC manifests for operator permissions in config/rbac/

### Internal Package Foundations

- [ ] T016 [P] Create MongoDB client interface and implementation in internal/mongodb/client.go
- [ ] T017 [P] Create Kubernetes resource manager utilities in internal/k8s/manager.go
- [ ] T018 [P] Create error types and utilities in internal/errors/errors.go
- [ ] T019 [P] Setup structured logging utilities in internal/logging/logger.go

### Test Infrastructure

- [ ] T020 [P] Setup envtest environment in controllers/suite_test.go
- [ ] T021 [P] Create KUTTL test configuration in tests/e2e/kuttl-test.yaml
- [ ] T022 [P] Setup GitHub Actions CI workflow in .github/workflows/test.yml

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Deploy MongoDB Instance (Priority: P1) 🎯 MVP

**Goal**: Platform operators can deploy MongoDB database instances on Kubernetes with specified configuration parameters

**Independent Test**: Request a new MongoDB deployment and verify the database is accessible and operational

### Unit Tests for User Story 1

- [ ] T023 [P] [US1] Unit tests for MongoDBInstance validation in api/v1alpha1/mongodbinstance_types_test.go
- [ ] T024 [P] [US1] Unit tests for StatefulSet generation in internal/k8s/statefulset_test.go
- [ ] T025 [P] [US1] Unit tests for Service generation in internal/k8s/service_test.go
- [ ] T026 [P] [US1] Unit tests for Secret generation in internal/k8s/secret_test.go
- [ ] T027 [P] [US1] Unit tests for PVC generation in internal/k8s/pvc_test.go

### Core Implementation for User Story 1

- [ ] T028 [P] [US1] Implement StatefulSet builder in internal/k8s/statefulset.go
- [ ] T029 [P] [US1] Implement Service builder in internal/k8s/service.go
- [ ] T030 [P] [US1] Implement PVC builder in internal/k8s/pvc.go
- [ ] T031 [P] [US1] Implement Secret builder with credential generation in internal/k8s/secret.go
- [ ] T032 [P] [US1] Implement ConfigMap builder for MongoDB config in internal/k8s/configmap.go
- [ ] T033 [US1] Create DeploymentManager interface implementation in controllers/deployment_manager.go
- [ ] T034 [US1] Implement MongoDBInstance reconciliation loop in controllers/mongodbinstance_controller.go
- [ ] T035 [US1] Add finalizer handling for cleanup in controllers/mongodbinstance_controller.go
- [ ] T036 [US1] Implement status updates with conditions in controllers/mongodbinstance_controller.go
- [ ] T037 [US1] Create sample CR manifest in config/samples/database_v1alpha1_mongodbinstance.yaml

### Integration Tests for User Story 1

- [ ] T038 [US1] Integration test for basic MongoDBInstance creation in controllers/mongodbinstance_controller_test.go
- [ ] T039 [US1] Integration test for resource ownership and garbage collection in controllers/mongodbinstance_controller_test.go
- [ ] T040 [US1] Integration test for status progression in controllers/mongodbinstance_controller_test.go

### E2E Tests for User Story 1

- [ ] T041 [US1] E2E test for MongoDB deployment lifecycle in tests/e2e/01-basic-deployment/
- [ ] T042 [US1] E2E test for MongoDB connectivity in tests/e2e/01-basic-deployment/01-test-connectivity.yaml

**Checkpoint**: At this point, User Story 1 (MongoDB deployment) should be fully functional and testable independently

---

## Phase 4: User Story 2 - Monitor Database Health (Priority: P2)

**Goal**: Platform operators can monitor the health and status of deployed MongoDB instances

**Independent Test**: Deploy a MongoDB instance and verify health metrics are accessible and accurate

### Unit Tests for User Story 2

- [ ] T043 [P] [US2] Unit tests for health check logic in controllers/health_checker_test.go
- [ ] T044 [P] [US2] Unit tests for MongoDB connection handling in internal/mongodb/client_test.go

### Implementation for User Story 2

- [ ] T045 [P] [US2] Implement MongoDB ping functionality in internal/mongodb/client.go
- [ ] T046 [P] [US2] Implement server status retrieval in internal/mongodb/client.go
- [ ] T047 [US2] Create HealthChecker interface implementation in controllers/health_checker.go
- [ ] T048 [US2] Add health check reconciliation to MongoDBInstance controller in controllers/mongodbinstance_controller.go
- [ ] T049 [US2] Implement health status conditions and messages in controllers/health_checker.go
- [ ] T050 [US2] Add event emission for health status changes in controllers/health_checker.go

### Integration Tests for User Story 2

- [ ] T051 [US2] Integration test for health check updates in controllers/health_checker_test.go
- [ ] T052 [US2] Integration test for health status transitions in controllers/mongodbinstance_controller_test.go

### E2E Tests for User Story 2

- [ ] T053 [US2] E2E test for health monitoring in tests/e2e/02-health-monitoring/

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently

---

## Phase 5: User Story 3 - Scale Database Resources (Priority: P3)

**Goal**: Platform operators can adjust resource allocation (storage, memory, CPU) for MongoDB instances without service interruption

**Independent Test**: Deploy a MongoDB instance, request a resource change, and verify new resources are applied while database remains available

### Unit Tests for User Story 3

- [ ] T054 [P] [US3] Unit tests for scaling validation in controllers/scaling_manager_test.go
- [ ] T055 [P] [US3] Unit tests for storage expansion logic in controllers/scaling_manager_test.go

### Implementation for User Story 3

- [ ] T056 [P] [US3] Implement resource scaling validation in controllers/scaling_manager.go
- [ ] T057 [P] [US3] Implement CPU/memory scaling in controllers/scaling_manager.go
- [ ] T058 [P] [US3] Implement storage expansion in controllers/scaling_manager.go
- [ ] T059 [US3] Create ScalingManager interface implementation in controllers/scaling_manager.go
- [ ] T060 [US3] Add scaling reconciliation to MongoDBInstance controller in controllers/mongodbinstance_controller.go
- [ ] T061 [US3] Add scaling events and status updates in controllers/scaling_manager.go

### Integration Tests for User Story 3

- [ ] T062 [US3] Integration test for resource scaling in controllers/scaling_manager_test.go
- [ ] T063 [US3] Integration test for storage expansion in controllers/scaling_manager_test.go
- [ ] T064 [US3] Integration test for idempotent scaling in controllers/mongodbinstance_controller_test.go

### E2E Tests for User Story 3

- [ ] T065 [US3] E2E test for scaling operations in tests/e2e/03-scaling/
- [ ] T066 [US3] E2E test for zero-downtime scaling in tests/e2e/03-scaling/02-assert-no-downtime.yaml

**Checkpoint**: User Stories 1, 2, AND 3 should all work independently

---

## Phase 6: User Story 4 - Backup and Restore (Priority: P3)

**Goal**: Platform operators can create backups of MongoDB instances and restore from backups to protect against data loss

**Independent Test**: Create a backup of a MongoDB instance with known data, simulate data loss, restore from backup, and verify data integrity

### Unit Tests for User Story 4

- [ ] T067 [P] [US4] Unit tests for backup Job generation in internal/backup/backup_test.go
- [ ] T068 [P] [US4] Unit tests for restore Job generation in internal/backup/restore_test.go
- [ ] T069 [P] [US4] Unit tests for retention policy in internal/backup/retention_test.go

### Implementation for User Story 4

- [ ] T070 [P] [US4] Implement backup Job creation in internal/backup/backup.go
- [ ] T071 [P] [US4] Implement restore Job creation in internal/backup/restore.go
- [ ] T072 [P] [US4] Implement backup retention logic in internal/backup/retention.go
- [ ] T073 [US4] Create BackupManager interface implementation in controllers/backup_manager.go
- [ ] T074 [US4] Implement MongoDBBackup reconciliation in controllers/mongodbbackup_controller.go
- [ ] T075 [US4] Implement MongoDBRestore reconciliation in controllers/mongodbrestore_controller.go
- [ ] T076 [US4] Add scheduled backup support to MongoDBInstance in controllers/mongodbinstance_controller.go
- [ ] T077 [US4] Create backup sample CRs in config/samples/database_v1alpha1_mongodbbackup.yaml
- [ ] T078 [US4] Create restore sample CRs in config/samples/database_v1alpha1_mongodbrestore.yaml

### Integration Tests for User Story 4

- [ ] T079 [US4] Integration test for backup creation in controllers/mongodbbackup_controller_test.go
- [ ] T080 [US4] Integration test for restore operation in controllers/mongodbrestore_controller_test.go
- [ ] T081 [US4] Integration test for backup retention in controllers/backup_manager_test.go

### E2E Tests for User Story 4

- [ ] T082 [US4] E2E test for manual backup in tests/e2e/04-backup/
- [ ] T083 [US4] E2E test for scheduled backups in tests/e2e/04-backup/02-scheduled.yaml
- [ ] T084 [US4] E2E test for restore operation in tests/e2e/04-backup/03-restore.yaml

**Checkpoint**: User Stories 1, 2, 3, AND 4 should all work independently

---

## Phase 7: User Story 5 - Remove Database Instance (Priority: P3)

**Goal**: Platform operators can decommission and remove MongoDB instances that are no longer needed to free up resources

**Independent Test**: Deploy a MongoDB instance, request its removal, and verify that all associated resources are cleaned up

### Unit Tests for User Story 5

- [ ] T085 [P] [US5] Unit tests for finalizer logic in controllers/mongodbinstance_controller_test.go
- [ ] T086 [P] [US5] Unit tests for resource cleanup in controllers/deployment_manager_test.go

### Implementation for User Story 5

- [ ] T087 [US5] Implement resource deletion in controllers/deployment_manager.go
- [ ] T088 [US5] Add backup check before deletion in controllers/mongodbinstance_controller.go
- [ ] T089 [US5] Implement deletion confirmation logic in controllers/mongodbinstance_controller.go
- [ ] T090 [US5] Add deletion events and logging in controllers/mongodbinstance_controller.go

### Integration Tests for User Story 5

- [ ] T091 [US5] Integration test for finalizer handling in controllers/mongodbinstance_controller_test.go
- [ ] T092 [US5] Integration test for resource cleanup in controllers/mongodbinstance_controller_test.go

### E2E Tests for User Story 5

- [ ] T093 [US5] E2E test for complete deletion in tests/e2e/05-deletion/

**Checkpoint**: All user stories should now be independently functional

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories and production readiness

### Documentation

- [ ] T094 [P] Create operator installation guide in docs/installation.md
- [ ] T095 [P] Create operator configuration guide in docs/configuration.md
- [ ] T096 [P] Create troubleshooting guide in docs/troubleshooting.md
- [ ] T097 [P] Update README.md with quickstart and examples

### Observability

- [ ] T098 [P] Add Prometheus metrics for operator in controllers/metrics.go
- [ ] T099 [P] Add Prometheus metrics for MongoDB instances in controllers/mongodbinstance_controller.go
- [ ] T100 [P] Create Grafana dashboard in config/grafana/dashboard.json
- [ ] T101 [P] Setup structured logging across all controllers

### Production Hardening

- [ ] T102 [P] Implement leader election in main.go
- [ ] T103 [P] Add graceful shutdown handling in main.go
- [ ] T104 [P] Configure resource limits for operator in config/manager/manager.yaml
- [ ] T105 [P] Create NetworkPolicy samples in config/samples/networkpolicy.yaml
- [ ] T106 [P] Create ResourceQuota samples in config/samples/resourcequota.yaml
- [ ] T107 [P] Add OpenTelemetry tracing in controllers/ (optional)

### Testing & Validation

- [ ] T108 Run complete E2E test suite per quickstart.md validation scenarios
- [ ] T109 Validate coverage targets (75% overall, 90% CRDs, 80% controllers)
- [ ] T110 [P] Performance testing with 50 instances in tests/performance/
- [ ] T111 [P] Chaos testing (pod failures, node failures) in tests/chaos/

### Deployment

- [ ] T112 Build and push container images to registry
- [ ] T113 [P] Create OLM bundle in bundle/
- [ ] T114 [P] Create Helm chart in charts/mongodb-operator/
- [ ] T115 Validate operator deployment on kind cluster
- [ ] T116 Validate operator deployment on production-like cluster

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3-7)**: All depend on Foundational phase completion
  - User stories can then proceed in parallel (if staffed)
  - Or sequentially in priority order (P1 → P2 → P3)
- **Polish (Phase 8)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P2)**: Can start after Foundational (Phase 2) - Integrates with US1 but independently testable
- **User Story 3 (P3)**: Can start after Foundational (Phase 2) - Extends US1 but independently testable
- **User Story 4 (P3)**: Can start after Foundational (Phase 2) - Uses US1 instances but independently testable
- **User Story 5 (P3)**: Can start after Foundational (Phase 2) - Uses US1 instances but independently testable

### Within Each User Story

- Unit tests (write first, ensure they FAIL before implementation)
- Core implementation (models, services, controllers)
- Integration tests
- E2E tests
- Story complete before moving to next priority

### Parallel Opportunities

**Phase 1 (Setup)**: T002, T003, T004, T005 can all run in parallel

**Phase 2 (Foundational)**:
- CRD Definitions: T006, T007, T008, T009 can run in parallel
- Controller Scaffolding: T011, T012, T013 can run in parallel
- Internal Packages: T016, T017, T018, T019 can run in parallel
- Test Infrastructure: T020, T021, T022 can run in parallel

**User Story 1 - US1 Unit Tests**: T023, T024, T025, T026, T027 can all run in parallel
**User Story 1 - US1 Builders**: T028, T029, T030, T031, T032 can all run in parallel

**User Story 2 - US2 Unit Tests**: T043, T044 can run in parallel
**User Story 2 - US2 Implementation**: T045, T046 can run in parallel

**User Story 3 - US3 Unit Tests**: T054, T055 can run in parallel
**User Story 3 - US3 Implementation**: T056, T057, T058 can run in parallel

**User Story 4 - US4 Unit Tests**: T067, T068, T069 can run in parallel
**User Story 4 - US4 Implementation**: T070, T071, T072 can run in parallel

**User Story 5 - US5 Unit Tests**: T085, T086 can run in parallel

**Phase 8 (Polish) - Documentation**: T094, T095, T096, T097 can run in parallel
**Phase 8 (Polish) - Observability**: T098, T099, T100, T101 can run in parallel
**Phase 8 (Polish) - Production**: T102-T107 can run in parallel
**Phase 8 (Polish) - Testing**: T110, T111 can run in parallel
**Phase 8 (Polish) - Deployment**: T113, T114 can run in parallel

---

## Parallel Example: User Story 1

```bash
# Launch all unit tests for User Story 1 together:
Task: T023 - Unit tests for MongoDBInstance validation
Task: T024 - Unit tests for StatefulSet generation
Task: T025 - Unit tests for Service generation
Task: T026 - Unit tests for Secret generation
Task: T027 - Unit tests for PVC generation

# Then launch all builders for User Story 1 together:
Task: T028 - Implement StatefulSet builder
Task: T029 - Implement Service builder
Task: T030 - Implement PVC builder
Task: T031 - Implement Secret builder
Task: T032 - Implement ConfigMap builder
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T005)
2. Complete Phase 2: Foundational (T006-T022) - CRITICAL - blocks all stories
3. Complete Phase 3: User Story 1 (T023-T042)
4. **STOP and VALIDATE**: Test User Story 1 independently
5. Deploy/demo basic MongoDB deployment capability

**Estimated effort**: 3-4 weeks for 1 developer

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready (Week 1-2)
2. Add User Story 1 → Test independently → Deploy/Demo (Week 3-4) - **MVP!**
3. Add User Story 2 → Test independently → Deploy/Demo (Week 5)
4. Add User Story 3 → Test independently → Deploy/Demo (Week 6-7)
5. Add User Story 4 → Test independently → Deploy/Demo (Week 8-9)
6. Add User Story 5 → Test independently → Deploy/Demo (Week 10)
7. Polish → Production ready (Week 11-12)

Each story adds value without breaking previous stories

### Parallel Team Strategy

With multiple developers:

1. **Week 1-2**: Team completes Setup + Foundational together
2. **Week 3-4**: Once Foundational is done:
   - Developer A: User Story 1 (T023-T042)
   - Developer B: User Story 2 (T043-T053)
   - Developer C: User Story 3 (T054-T066)
3. **Week 5-6**:
   - Developer A: User Story 4 (T067-T084)
   - Developer B: User Story 5 (T085-T093)
   - Developer C: Polish & Observability (T094-T107)
4. **Week 7**: All developers on testing and production hardening (T108-T116)

Stories complete and integrate independently

---

## Notes

- [P] tasks = different files, no dependencies - can run concurrently
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Testing strategy per research.md: 60% unit, 30% integration, 10% E2E
- Coverage targets: 75% overall, 90% CRDs, 80% controllers
- Follow TDD: Write tests first, ensure they fail, then implement
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- MVP = User Story 1 only (basic MongoDB deployment)
- v2 features: Replica sets, volume snapshots, enhanced scale (100+ instances)

---

## Task Count Summary

- **Phase 1 (Setup)**: 5 tasks
- **Phase 2 (Foundational)**: 17 tasks
- **Phase 3 (US1 - Deploy)**: 20 tasks
- **Phase 4 (US2 - Monitor)**: 11 tasks
- **Phase 5 (US3 - Scale)**: 13 tasks
- **Phase 6 (US4 - Backup)**: 18 tasks
- **Phase 7 (US5 - Remove)**: 9 tasks
- **Phase 8 (Polish)**: 23 tasks

**Total: 116 tasks**

**MVP (US1 only)**: 42 tasks (Setup + Foundational + US1)
**Production Ready (All stories)**: 116 tasks
