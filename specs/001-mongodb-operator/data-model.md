# Data Model: MongoDB Operator

**Feature**: MongoDB Operator Deployment
**Branch**: `001-mongodb-operator`
**Date**: 2025-10-23

## Overview

This document defines the data entities for the MongoDB Kubernetes operator. The operator uses Custom Resource Definitions (CRDs) to represent MongoDB instances and related resources in a Kubernetes-native way.

## Core Entities

### 1. MongoDBInstance (Custom Resource)

The primary CRD representing a MongoDB database instance managed by the operator.

#### Kubernetes Metadata
- **API Group**: `database.vteam.io`
- **API Version**: `v1alpha1`
- **Kind**: `MongoDBInstance`
- **Namespaced**: Yes
- **Scope**: Cluster-wide (can be deployed in any namespace)

#### Spec Fields

| Field | Type | Required | Default | Description | Validation Rules |
|-------|------|----------|---------|-------------|------------------|
| `version` | string | Yes | - | MongoDB version (e.g., "7.0", "6.0") | Must match supported versions list |
| `storage.size` | string | Yes | - | Persistent volume size (e.g., "10Gi", "100Gi") | Must be valid Kubernetes quantity, minimum 1Gi |
| `storage.storageClassName` | string | No | (cluster default) | Kubernetes StorageClass name | Must reference existing StorageClass |
| `resources.requests.memory` | string | Yes | - | Memory request (e.g., "2Gi") | Must be valid Kubernetes quantity |
| `resources.requests.cpu` | string | Yes | - | CPU request (e.g., "1000m", "1") | Must be valid Kubernetes quantity |
| `resources.limits.memory` | string | No | 2x requests | Memory limit | Must be >= requests |
| `resources.limits.cpu` | string | No | 2x requests | CPU limit | Must be >= requests |
| `backup.enabled` | boolean | No | false | Enable automated backups | - |
| `backup.schedule` | string | No* | - | Cron schedule for backups | Required if backup.enabled=true, must be valid cron |
| `backup.retention` | integer | No | 7 | Number of backups to retain | Minimum 1, maximum 365 |
| `backup.destination.type` | string | No* | - | Backup storage type (s3, gcs, azure) | Required if backup.enabled=true |
| `backup.destination.bucket` | string | No* | - | Storage bucket name | Required if backup.enabled=true |
| `backup.destination.secretRef` | string | No* | - | Secret containing storage credentials | Required if backup.enabled=true, must reference existing Secret |
| `network.serviceType` | string | No | ClusterIP | Kubernetes Service type | Must be ClusterIP, NodePort, or LoadBalancer |
| `network.port` | integer | No | 27017 | MongoDB connection port | 1024-65535 |

#### Status Fields (Read-Only)

| Field | Type | Description |
|-------|------|-------------|
| `phase` | string | Current lifecycle phase: Pending, Creating, Running, Scaling, Failed, Deleting |
| `conditions` | []Condition | Detailed status conditions (Ready, StorageReady, NetworkReady) |
| `health.status` | string | Health status: Healthy, Degraded, Unavailable |
| `health.lastCheckTime` | timestamp | Last health check timestamp |
| `health.message` | string | Human-readable health status message |
| `connection.hostname` | string | Connection hostname (Service DNS name) |
| `connection.port` | integer | Connection port |
| `connection.credentialsSecretRef` | string | Secret name containing connection credentials |
| `resources.actualStorage` | string | Current storage allocation |
| `resources.usedStorage` | string | Storage currently in use |
| `backup.lastBackupTime` | timestamp | Timestamp of most recent successful backup |
| `backup.lastBackupStatus` | string | Status of most recent backup (Success, Failed, InProgress) |
| `observedGeneration` | integer | Last observed spec generation (for reconciliation tracking) |

#### Example Custom Resource

```yaml
apiVersion: database.vteam.io/v1alpha1
kind: MongoDBInstance
metadata:
  name: app-mongodb
  namespace: production
spec:
  version: "7.0"
  storage:
    size: "50Gi"
    storageClassName: "gp3"
  resources:
    requests:
      memory: "4Gi"
      cpu: "2000m"
    limits:
      memory: "8Gi"
      cpu: "4000m"
  backup:
    enabled: true
    schedule: "0 2 * * *"  # Daily at 2 AM
    retention: 14
    destination:
      type: s3
      bucket: "mongodb-backups"
      secretRef: "s3-credentials"
  network:
    serviceType: ClusterIP
    port: 27017
status:
  phase: Running
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2025-10-23T14:30:00Z"
    reason: InstanceHealthy
    message: "MongoDB instance is running and healthy"
  - type: StorageReady
    status: "True"
    lastTransitionTime: "2025-10-23T14:25:00Z"
    reason: PVCBound
    message: "Persistent volume claim bound successfully"
  - type: NetworkReady
    status: "True"
    lastTransitionTime: "2025-10-23T14:26:00Z"
    reason: ServiceCreated
    message: "Service created and endpoints ready"
  health:
    status: Healthy
    lastCheckTime: "2025-10-23T14:55:00Z"
    message: "All health checks passing"
  connection:
    hostname: "app-mongodb.production.svc.cluster.local"
    port: 27017
    credentialsSecretRef: "app-mongodb-credentials"
  resources:
    actualStorage: "50Gi"
    usedStorage: "12.3Gi"
  backup:
    lastBackupTime: "2025-10-23T02:00:00Z"
    lastBackupStatus: Success
  observedGeneration: 3
```

### 2. MongoDBBackup (Custom Resource)

Represents a point-in-time backup of a MongoDB instance.

#### Kubernetes Metadata
- **API Group**: `database.vteam.io`
- **API Version**: `v1alpha1`
- **Kind**: `MongoDBBackup`
- **Namespaced**: Yes
- **Scope**: Cluster-wide

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `instanceRef.name` | string | Yes | Name of MongoDBInstance to backup |
| `instanceRef.namespace` | string | No | Namespace of instance (defaults to backup namespace) |
| `destination.type` | string | Yes | Backup storage type (s3, gcs, azure) |
| `destination.bucket` | string | Yes | Storage bucket name |
| `destination.path` | string | No | Path prefix within bucket |
| `destination.secretRef` | string | Yes | Secret containing storage credentials |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `phase` | string | Backup phase: Pending, InProgress, Completed, Failed |
| `startTime` | timestamp | Backup start time |
| `completionTime` | timestamp | Backup completion time |
| `size` | string | Compressed backup size |
| `storageLocation` | string | Full path to backup in storage |
| `message` | string | Human-readable status message |

#### Example Custom Resource

```yaml
apiVersion: database.vteam.io/v1alpha1
kind: MongoDBBackup
metadata:
  name: app-mongodb-backup-20251023
  namespace: production
spec:
  instanceRef:
    name: app-mongodb
    namespace: production
  destination:
    type: s3
    bucket: "mongodb-backups"
    path: "production/app-mongodb"
    secretRef: "s3-credentials"
status:
  phase: Completed
  startTime: "2025-10-23T02:00:00Z"
  completionTime: "2025-10-23T02:12:34Z"
  size: "3.2Gi"
  storageLocation: "s3://mongodb-backups/production/app-mongodb/backup-20251023-020000.tar.gz"
  message: "Backup completed successfully"
```

### 3. MongoDBRestore (Custom Resource)

Represents a restore operation from a backup.

#### Kubernetes Metadata
- **API Group**: `database.vteam.io`
- **API Version**: `v1alpha1`
- **Kind**: `MongoDBRestore`
- **Namespaced**: Yes
- **Scope**: Cluster-wide

#### Spec Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `targetInstanceRef.name` | string | Yes | Name of MongoDBInstance to restore to |
| `targetInstanceRef.namespace` | string | No | Namespace of instance |
| `backupRef.name` | string | Yes | Name of MongoDBBackup to restore from |
| `backupRef.namespace` | string | No | Namespace of backup |

#### Status Fields

| Field | Type | Description |
|-------|------|-------------|
| `phase` | string | Restore phase: Pending, InProgress, Completed, Failed |
| `startTime` | timestamp | Restore start time |
| `completionTime` | timestamp | Restore completion time |
| `message` | string | Human-readable status message |

## Kubernetes Native Resources

The operator creates and manages standard Kubernetes resources for each MongoDBInstance:

### StatefulSet
- **Purpose**: Manages the MongoDB pod(s)
- **Replicas**: 1 (standalone mode for MVP)
- **Pod Template**: MongoDB container with configured resources
- **Volume Mount**: PersistentVolumeClaim for data directory
- **Owner Reference**: MongoDBInstance (for garbage collection)

### PersistentVolumeClaim
- **Purpose**: Requests persistent storage for MongoDB data
- **Storage Class**: From MongoDBInstance spec or cluster default
- **Access Mode**: ReadWriteOnce
- **Capacity**: From MongoDBInstance spec.storage.size
- **Owner Reference**: MongoDBInstance

### Service
- **Purpose**: Provides stable network endpoint for MongoDB
- **Type**: From MongoDBInstance spec.network.serviceType
- **Port**: From MongoDBInstance spec.network.port (default 27017)
- **Selector**: Matches StatefulSet pod labels
- **Owner Reference**: MongoDBInstance

### Secret (Credentials)
- **Purpose**: Stores MongoDB connection credentials
- **Data Keys**:
  - `username`: MongoDB admin username
  - `password`: Generated strong password
  - `connectionString`: Full MongoDB connection URI
- **Owner Reference**: MongoDBInstance

### ConfigMap (MongoDB Configuration)
- **Purpose**: Stores MongoDB server configuration
- **Data Keys**:
  - `mongod.conf`: MongoDB configuration file
- **Owner Reference**: MongoDBInstance

## State Machine: MongoDBInstance Lifecycle

```
┌─────────┐
│ Pending │ Initial state when CR is created
└────┬────┘
     │
     v
┌─────────┐
│Creating │ Operator creating StatefulSet, PVC, Service, Secrets
└────┬────┘
     │
     ├─ Success ──> ┌─────────┐
     │              │ Running │ MongoDB is operational
     │              └────┬────┘
     │                   │
     │                   ├─ Scale Request ──> ┌─────────┐
     │                   │                     │ Scaling │
     │                   │                     └────┬────┘
     │                   │                          │
     │                   │ <────────────────────────┘
     │                   │
     │                   ├─ Delete Request ──> ┌──────────┐
     │                   │                      │ Deleting │
     │                   │                      └────┬─────┘
     │                   │                           │
     │                   │                           v
     │                   │                      (Resource removed)
     │                   │
     │                   └─ Health Failure ──> ┌──────────┐
     │                                          │ Degraded │
     │                                          └────┬─────┘
     │                                               │
     │                                               └─ Recovers ──> Running
     │
     └─ Failure ──> ┌────────┐
                    │ Failed │ Permanent failure, manual intervention needed
                    └────────┘
```

## Health Status Transitions

```
Healthy: All health checks passing
   ↕
Degraded: Some health checks failing, instance partially functional
   ↕
Unavailable: Cannot connect to MongoDB, instance not functional
```

## Validation Rules

### Creation Validation
1. **Storage size**: Must be >= 1Gi
2. **Memory requests**: Must be >= 512Mi (MongoDB minimum)
3. **MongoDB version**: Must be in supported versions list (6.0, 7.0)
4. **Backup schedule**: If enabled, must be valid cron expression
5. **Storage class**: If specified, must exist in cluster
6. **Secret references**: Must exist in same namespace

### Update Validation
1. **Storage size**: Can only increase (no shrinking)
2. **MongoDB version**: Can only upgrade (no downgrades)
3. **Resource requests**: Changes trigger pod restart
4. **Immutable fields**: `metadata.name`, `spec.storage.storageClassName`

### Deletion Validation
1. **Finalizers**: Operator adds finalizer to ensure cleanup
2. **Backup check**: Warn if no recent backup exists
3. **Cleanup**: Remove all owned resources (StatefulSet, PVC, Service, Secrets)

## Relationships

```
MongoDBInstance (1)
  │
  ├─ owns ──> StatefulSet (1)
  │             └─ manages ──> Pod(s) (1 in MVP)
  │
  ├─ owns ──> PersistentVolumeClaim (1)
  │             └─ binds ──> PersistentVolume (1)
  │
  ├─ owns ──> Service (1)
  │
  ├─ owns ──> Secret (1 for credentials)
  │
  ├─ owns ──> ConfigMap (1 for mongod.conf)
  │
  └─ references <─── MongoDBBackup (0..*)
                       │
                       └─ referenced by ──> MongoDBRestore (0..*)
```

## Storage Considerations

Based on research decisions:

### Storage Classes (Platform-Specific)
- **AWS EKS**: `gp3` (default), `io2` (high IOPS)
- **Azure AKS**: `managed-premium` (SSD)
- **GCP GKE**: `pd-ssd` (standard), `pd-extreme` (high performance)
- **OpenShift**: `rook-ceph-block`, `local-path`

### Performance Targets
- **IOPS**: Minimum 3000 (production workloads)
- **Throughput**: 250 MB/s for bulk operations
- **Latency**: <5ms for 99th percentile

### Backup Storage Format
- **Tool**: mongodump/mongorestore
- **Compression**: gzip (70-80% compression ratio)
- **Format**: BSON dump + metadata JSON
- **Naming**: `{instance}-{timestamp}.tar.gz`

## Index Strategy

Not applicable - this operator manages MongoDB instances but doesn't define MongoDB collections or indexes. Application teams define their own database schemas.

## Data Retention

### Backup Retention
- Configurable per MongoDBInstance via `spec.backup.retention`
- Default: 7 days
- Automatic cleanup of old backups by operator
- Implemented via CronJob checking backup age

### Log Retention
- MongoDB logs: 7 days (via logrotate in container)
- Operator logs: 30 days (Kubernetes default)

## Security Considerations

### Credentials Management
- **Generation**: Operator generates strong passwords (32 characters, alphanumeric + symbols)
- **Storage**: Kubernetes Secrets with base64 encoding
- **Rotation**: Not implemented in MVP (v2 feature)
- **Access**: RBAC controls Secret access

### Network Isolation
- **Default**: ClusterIP service (cluster-internal only)
- **Production**: Use NetworkPolicies to restrict access per namespace
- **External Access**: LoadBalancer or Ingress (operator doesn't create, user managed)

### Data Encryption
- **At Rest**: Depends on PV storage class encryption settings
- **In Transit**: TLS for MongoDB connections (v2 feature)
- **Backups**: Encrypted by storage provider (S3 SSE, Azure encryption)

## Observability

### Metrics (Future Enhancement)
Each MongoDBInstance should expose:
- Connection count
- Storage usage percentage
- Query operations per second
- Replication lag (v2 with replica sets)

### Logs
- Operator logs: Controller reconciliation events
- MongoDB logs: Forwarded to stdout for cluster logging
- Backup/restore logs: Job logs in Kubernetes

### Events
Operator emits Kubernetes Events for:
- Instance creation progress
- Health status changes
- Scaling operations
- Backup/restore status
- Error conditions
