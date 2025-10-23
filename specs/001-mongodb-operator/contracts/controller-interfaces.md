# Controller Interfaces

**Feature**: MongoDB Operator Deployment
**Branch**: `001-mongodb-operator`
**Date**: 2025-10-23

## Overview

This document defines the internal controller interfaces and contracts for the MongoDB operator. These interfaces define the reconciliation logic and component interactions.

## Core Controller Interface

### MongoDBInstanceReconciler

The primary controller that watches MongoDBInstance resources and orchestrates reconciliation.

```go
package controllers

import (
    "context"
    databasev1alpha1 "github.com/vteam/mongodb-operator/api/v1alpha1"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"
)

// MongoDBInstanceReconciler reconciles a MongoDBInstance object
type MongoDBInstanceReconciler struct {
    client.Client
    Scheme          *runtime.Scheme
    Recorder        record.EventRecorder
    DeploymentMgr   DeploymentManager
    HealthChecker   HealthChecker
    ScalingMgr      ScalingManager
    BackupMgr       BackupManager
}

// Reconcile implements the reconciliation loop
// +kubebuilder:rbac:groups=database.vteam.io,resources=mongodbinstances,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=database.vteam.io,resources=mongodbinstances/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=database.vteam.io,resources=mongodbinstances/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services;persistentvolumeclaims;secrets;configmaps,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=events,verbs=create;patch
func (r *MongoDBInstanceReconciler) Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error)
```

**Reconciliation Logic**:
1. Fetch MongoDBInstance resource
2. Check if resource is being deleted (finalizer handling)
3. Validate spec (version, resources, storage)
4. Ensure infrastructure resources (StatefulSet, PVC, Service, Secrets)
5. Check health status
6. Handle scaling requests
7. Update status
8. Requeue for health checks (60 second interval)

**Return Values**:
- `(reconcile.Result{}, nil)`: Success, no requeue
- `(reconcile.Result{RequeueAfter: 60*time.Second}, nil)`: Success, requeue for health check
- `(reconcile.Result{}, err)`: Error, exponential backoff requeue
- `(reconcile.Result{Requeue: true}, nil)`: Immediate requeue (transient state)

## Component Interfaces

### DeploymentManager

Manages creation and updates of Kubernetes resources for MongoDB instances.

```go
type DeploymentManager interface {
    // EnsureStatefulSet creates or updates the MongoDB StatefulSet
    // Returns true if created/updated, false if no changes needed
    EnsureStatefulSet(ctx context.Context, instance *databasev1alpha1.MongoDBInstance) (bool, error)

    // EnsurePVC creates the PersistentVolumeClaim if it doesn't exist
    // Returns true if created, false if already exists
    EnsurePVC(ctx context.Context, instance *databasev1alpha1.MongoDBInstance) (bool, error)

    // EnsureService creates or updates the MongoDB Service
    // Returns true if created/updated, false if no changes needed
    EnsureService(ctx context.Context, instance *databasev1alpha1.MongoDBInstance) (bool, error)

    // EnsureSecrets creates the credentials Secret if it doesn't exist
    // Returns the Secret name and whether it was created
    EnsureSecrets(ctx context.Context, instance *databasev1alpha1.MongoDBInstance) (string, bool, error)

    // EnsureConfigMap creates or updates the MongoDB configuration ConfigMap
    // Returns true if created/updated, false if no changes needed
    EnsureConfigMap(ctx context.Context, instance *databasev1alpha1.MongoDBInstance) (bool, error)

    // DeleteResources cleans up all resources owned by the MongoDBInstance
    // Used during finalizer processing
    DeleteResources(ctx context.Context, instance *databasev1alpha1.MongoDBInstance) error
}
```

**Implementation Notes**:
- All methods must be idempotent (can be called multiple times safely)
- Use OwnerReferences for garbage collection
- Return structured errors with context for debugging
- Emit Kubernetes Events for user-visible operations

### HealthChecker

Monitors MongoDB instance health and updates status.

```go
type HealthChecker interface {
    // CheckHealth performs health checks on the MongoDB instance
    // Returns health status (Healthy/Degraded/Unavailable) and optional error
    CheckHealth(ctx context.Context, instance *databasev1alpha1.MongoDBInstance) (HealthStatus, error)

    // GetConnectionInfo retrieves connection details for health checking
    // Returns hostname, port, credentials for MongoDB connection
    GetConnectionInfo(ctx context.Context, instance *databasev1alpha1.MongoDBInstance) (ConnectionInfo, error)

    // UpdateHealthStatus updates the instance status with health check results
    UpdateHealthStatus(ctx context.Context, instance *databasev1alpha1.MongoDBInstance, status HealthStatus) error
}

type HealthStatus struct {
    Status       string    // "Healthy", "Degraded", "Unavailable"
    Message      string    // Human-readable description
    LastCheckTime time.Time
}

type ConnectionInfo struct {
    Hostname string
    Port     int32
    Username string
    Password string
}
```

**Health Check Criteria**:
- **Healthy**: MongoDB connection succeeds, ping command responds, CPU/memory within limits
- **Degraded**: Connection succeeds but slow, high resource usage (>90%), or non-critical errors
- **Unavailable**: Cannot connect, authentication fails, or critical errors

**Timeout**: 10 seconds per health check
**Frequency**: Every 60 seconds (controlled by reconciliation loop)

### ScalingManager

Handles resource scaling operations (storage, CPU, memory).

```go
type ScalingManager interface {
    // ScaleResources applies resource changes to the StatefulSet
    // Handles both vertical scaling (CPU/memory) and storage expansion
    // Returns true if scaling operation initiated, false if no changes needed
    ScaleResources(ctx context.Context, instance *databasev1alpha1.MongoDBInstance, current *appsv1.StatefulSet) (bool, error)

    // ExpandStorage increases PVC storage capacity
    // Returns true if expansion initiated, false if not supported or no change
    ExpandStorage(ctx context.Context, instance *databasev1alpha1.MongoDBInstance, currentPVC *corev1.PersistentVolumeClaim) (bool, error)

    // ValidateScaling checks if requested scaling is valid and safe
    // Returns error if scaling would violate constraints
    ValidateScaling(ctx context.Context, instance *databasev1alpha1.MongoDBInstance, current *appsv1.StatefulSet) error
}
```

**Scaling Constraints**:
- Storage can only increase (no shrinking)
- MongoDB version can only upgrade (no downgrades)
- Resource changes trigger pod restart (unavoidable)
- Storage expansion requires storage class with `allowVolumeExpansion: true`

**Validation Rules**:
- New storage size must be > current size
- Memory must be >= 512Mi (MongoDB minimum)
- CPU must be > 0
- Limits must be >= requests

### BackupManager

Manages backup and restore operations.

```go
type BackupManager interface {
    // CreateBackup initiates a backup job for the MongoDBInstance
    // Creates a MongoDBBackup custom resource and corresponding Job
    CreateBackup(ctx context.Context, instance *databasev1alpha1.MongoDBInstance, schedule string) error

    // ReconcileBackup handles the backup reconciliation loop
    // Updates backup status based on Job status
    ReconcileBackup(ctx context.Context, backup *databasev1alpha1.MongoDBBackup) (reconcile.Result, error)

    // CreateRestore initiates a restore operation
    // Creates a Job to restore data from backup
    CreateRestore(ctx context.Context, restore *databasev1alpha1.MongoDBRestore) error

    // ReconcileRestore handles the restore reconciliation loop
    // Updates restore status based on Job status
    ReconcileRestore(ctx context.Context, restore *databasev1alpha1.MongoDBRestore) (reconcile.Result, error)

    // CleanupOldBackups removes backups older than retention period
    CleanupOldBackups(ctx context.Context, instance *databasev1alpha1.MongoDBInstance) error
}
```

**Backup Process**:
1. Create Kubernetes Job running mongodump container
2. Job mounts credentials Secret
3. mongodump connects to MongoDB, exports data
4. Compress and upload to S3/GCS/Azure
5. Update MongoDBBackup status with size and location
6. Cleanup local files

**Restore Process**:
1. Validate target instance exists and is in Running phase
2. Create Kubernetes Job running mongorestore container
3. Job downloads backup from storage
4. Decompress and restore to MongoDB
5. Update MongoDBRestore status
6. Cleanup local files

## Internal Package Interfaces

### MongoDB Client

```go
package mongodb

type Client interface {
    // Connect establishes connection to MongoDB instance
    Connect(ctx context.Context, connectionString string) error

    // Ping tests MongoDB connectivity
    Ping(ctx context.Context) error

    // GetServerStatus retrieves MongoDB server status
    GetServerStatus(ctx context.Context) (*ServerStatus, error)

    // Close closes the MongoDB connection
    Close(ctx context.Context) error
}

type ServerStatus struct {
    Version      string
    Uptime       int64
    Connections  int32
    MemoryUsage  int64
}
```

### Kubernetes Resource Manager

```go
package k8s

type ResourceManager interface {
    // StatefulSet operations
    GetStatefulSet(ctx context.Context, namespace, name string) (*appsv1.StatefulSet, error)
    CreateStatefulSet(ctx context.Context, sts *appsv1.StatefulSet) error
    UpdateStatefulSet(ctx context.Context, sts *appsv1.StatefulSet) error

    // PVC operations
    GetPVC(ctx context.Context, namespace, name string) (*corev1.PersistentVolumeClaim, error)
    CreatePVC(ctx context.Context, pvc *corev1.PersistentVolumeClaim) error

    // Service operations
    GetService(ctx context.Context, namespace, name string) (*corev1.Service, error)
    CreateService(ctx context.Context, svc *corev1.Service) error
    UpdateService(ctx context.Context, svc *corev1.Service) error

    // Secret operations
    GetSecret(ctx context.Context, namespace, name string) (*corev1.Secret, error)
    CreateSecret(ctx context.Context, secret *corev1.Secret) error

    // ConfigMap operations
    GetConfigMap(ctx context.Context, namespace, name string) (*corev1.ConfigMap, error)
    CreateConfigMap(ctx context.Context, cm *corev1.ConfigMap) error
    UpdateConfigMap(ctx context.Context, cm *corev1.ConfigMap) error

    // Pod operations (for health checking)
    ListPods(ctx context.Context, namespace string, labels map[string]string) (*corev1.PodList, error)
}
```

## Event Types

The operator emits Kubernetes Events for user-visible operations:

| Event Type | Reason | Message Template |
|------------|--------|------------------|
| Normal | Created | "Created MongoDB instance %s" |
| Normal | StatefulSetReady | "StatefulSet %s is ready" |
| Normal | ServiceCreated | "Service %s created" |
| Normal | StorageProvisioned | "Storage provisioned: %s" |
| Normal | HealthCheckPassed | "Health check passed" |
| Normal | ScalingStarted | "Scaling resources: %s" |
| Normal | ScalingCompleted | "Resource scaling completed" |
| Normal | BackupStarted | "Backup initiated: %s" |
| Normal | BackupCompleted | "Backup completed: %s" |
| Normal | RestoreStarted | "Restore initiated from backup %s" |
| Normal | RestoreCompleted | "Restore completed" |
| Warning | CreationFailed | "Failed to create resources: %s" |
| Warning | HealthCheckFailed | "Health check failed: %s" |
| Warning | ScalingFailed | "Scaling failed: %s" |
| Warning | BackupFailed | "Backup failed: %s" |
| Warning | RestoreFailed | "Restore failed: %s" |
| Warning | DeletionBlocked | "Deletion requires confirmation (no recent backup)" |

## Error Handling

### Error Types

```go
type OperatorError struct {
    Type    ErrorType
    Message string
    Cause   error
}

type ErrorType string

const (
    ErrorTypeValidation     ErrorType = "Validation"
    ErrorTypeResourceCreate ErrorType = "ResourceCreate"
    ErrorTypeResourceUpdate ErrorType = "ResourceUpdate"
    ErrorTypeConnection     ErrorType = "Connection"
    ErrorTypeHealthCheck    ErrorType = "HealthCheck"
    ErrorTypeScaling        ErrorType = "Scaling"
    ErrorTypeBackup         ErrorType = "Backup"
    ErrorTypeRestore        ErrorType = "Restore"
)
```

### Retry Logic

- **Transient errors** (connection failures, API server unavailable): Exponential backoff (1s, 2s, 4s, 8s, 16s, max 5 minutes)
- **Validation errors**: No retry (requires user intervention)
- **Resource conflicts**: Immediate requeue (likely concurrent modification)
- **Health check failures**: Continue reconciling (don't block other operations)

## Finalizer Logic

Finalizer: `database.vteam.io/mongodb-finalizer`

**Purpose**: Ensure clean resource deletion before removing MongoDBInstance CR

**Process**:
1. Operator adds finalizer when creating MongoDBInstance
2. On deletion request, Kubernetes marks resource for deletion but doesn't remove it
3. Operator detects deletion timestamp
4. Operator performs cleanup:
   - Check for recent backup (warn if none exists)
   - Delete StatefulSet
   - Delete Service
   - Delete Secrets
   - Delete ConfigMap
   - Optionally delete PVC (based on retention policy)
5. Operator removes finalizer
6. Kubernetes removes MongoDBInstance CR

## Status Update Strategy

**Approach**: Separate status updates from spec reconciliation

**Pattern**:
```go
// 1. Reconcile resources
if err := r.reconcileResources(ctx, instance); err != nil {
    // Update status with error
    r.updateStatusWithError(ctx, instance, err)
    return reconcile.Result{}, err
}

// 2. Check health
health, err := r.HealthChecker.CheckHealth(ctx, instance)
if err != nil {
    r.updateStatusWithError(ctx, instance, err)
    return reconcile.Result{RequeueAfter: 60*time.Second}, nil
}

// 3. Update status with current state
r.updateStatus(ctx, instance, health)
return reconcile.Result{RequeueAfter: 60*time.Second}, nil
```

**Status Update Isolation**: Status updates use `Status()` subresource to avoid conflicts with spec updates

## Reconciliation Performance

**Target Performance**:
- Initial deployment: <10 minutes (mostly waiting for StatefulSet)
- Reconciliation loop: <1 second (when no changes needed)
- Health check: <10 seconds
- Scaling operation: <5 minutes (CPU/memory), <30 minutes (storage)

**Optimization Strategies**:
- Cache client reads (controller-runtime cache)
- Avoid unnecessary API calls (check cache first)
- Batch status updates (don't update on every check)
- Use informers for event-driven reconciliation

## Testing Contracts

Each interface must have:
- **Unit tests**: Mock implementations, test business logic
- **Integration tests**: Real Kubernetes API (envtest), test resource creation
- **E2E tests**: Full operator deployment, test end-to-end scenarios

**Test Coverage Targets**:
- DeploymentManager: 80%
- HealthChecker: 80%
- ScalingManager: 80%
- BackupManager: 75%
- Reconciler: 80%
