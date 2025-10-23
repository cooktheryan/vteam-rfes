# vteam-rfes Development Guidelines

Auto-generated from all feature plans. Last updated: 2025-10-23

## Active Technologies
- **Language**: Go (with Operator SDK/Kubebuilder)
- **Framework**: Kubernetes Operator SDK, controller-runtime
- **Platform**: Kubernetes 1.24+
- **Database**: MongoDB (versions 6.0, 7.0)
- **Testing**: Go testing (testify), envtest, KUTTL
- **Storage**: Kubernetes PersistentVolumes, S3/GCS/Azure for backups

## Project Structure
```
mongodb-operator/
├── api/v1alpha1/          # CRD definitions (MongoDBInstance, MongoDBBackup, MongoDBRestore)
├── controllers/           # Reconciliation logic
├── internal/
│   ├── mongodb/          # MongoDB client and configuration
│   ├── k8s/              # Kubernetes resource management
│   └── backup/           # Backup/restore logic
├── config/               # Kubernetes manifests (CRDs, RBAC, deployment)
└── tests/
    ├── e2e/              # End-to-end tests with KUTTL
    ├── integration/      # Integration tests with envtest
    └── unit/             # Unit tests with testify
```

## Development Workflow

### Initial Setup
```bash
# Install kubebuilder/operator-sdk
# Scaffold operator project
kubebuilder init --domain vteam.io --repo github.com/vteam/mongodb-operator

# Create CRDs
kubebuilder create api --group database --version v1alpha1 --kind MongoDBInstance
kubebuilder create api --group database --version v1alpha1 --kind MongoDBBackup
kubebuilder create api --group database --version v1alpha1 --kind MongoDBRestore

# Generate manifests
make manifests

# Install CRDs
make install
```

### Testing
```bash
# Unit tests
go test ./... -v

# Integration tests with envtest
make test

# E2E tests with KUTTL
make test-e2e

# Run locally
make run
```

### Building and Deployment
```bash
# Build operator
make build

# Build and push Docker image
make docker-build docker-push IMG=your-registry/mongodb-operator:tag

# Deploy to cluster
make deploy IMG=your-registry/mongodb-operator:tag
```

## Commands
```bash
# Create MongoDB instance
kubectl apply -f config/samples/database_v1alpha1_mongodbinstance.yaml

# Check status
kubectl get mongodbinstance
kubectl describe mongodbinstance <name>

# Manual backup
kubectl apply -f config/samples/database_v1alpha1_mongodbbackup.yaml

# Restore
kubectl apply -f config/samples/database_v1alpha1_mongodbrestore.yaml

# View logs
kubectl logs -n mongodb-operator-system deployment/mongodb-operator-controller-manager
```

## Code Style
- **Go Standards**: Follow Go conventions, use gofmt
- **Controller Pattern**: Idempotent reconciliation, use controller-runtime patterns
- **Error Handling**: Return structured errors, use errors.Wrap for context
- **Logging**: Use logr for structured logging
- **Comments**: Document exported functions, use godoc format
- **Testing**: Table-driven tests, use testify/assert for assertions

## Recent Changes
- 001-mongodb-operator: Added

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
