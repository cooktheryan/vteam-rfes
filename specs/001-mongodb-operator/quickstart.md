# MongoDB Operator Quickstart Guide

**Feature**: MongoDB Operator Deployment
**Branch**: `001-mongodb-operator`
**Date**: 2025-10-23

## Overview

This quickstart guide walks you through deploying and using the MongoDB Operator to manage MongoDB database instances on Kubernetes.

## Prerequisites

- Kubernetes cluster 1.24+ (kind, minikube, EKS, AKS, GKE, or OpenShift)
- `kubectl` configured to access your cluster
- Cluster admin permissions (for installing CRDs and operator)
- At least 4GB memory and 2 CPU cores available in cluster
- Default StorageClass configured (or specify custom storage class)

**Verify Prerequisites**:
```bash
# Check Kubernetes version
kubectl version --short

# Check available resources
kubectl top nodes

# Check default storage class
kubectl get storageclass
```

## Installation

### Step 1: Install the MongoDB Operator

```bash
# Install CRDs
kubectl apply -f https://github.com/vteam/mongodb-operator/releases/latest/download/crds.yaml

# Install operator
kubectl apply -f https://github.com/vteam/mongodb-operator/releases/latest/download/operator.yaml

# Verify operator is running
kubectl get pods -n mongodb-operator-system
```

Expected output:
```
NAME                                          READY   STATUS    RESTARTS   AGE
mongodb-operator-controller-manager-xxx       2/2     Running   0          30s
```

### Step 2: Deploy Your First MongoDB Instance

Create a file named `my-mongodb.yaml`:

```yaml
apiVersion: database.vteam.io/v1alpha1
kind: MongoDBInstance
metadata:
  name: my-first-mongodb
  namespace: default
spec:
  version: "7.0"
  storage:
    size: "10Gi"
  resources:
    requests:
      memory: "2Gi"
      cpu: "1000m"
```

Apply the configuration:
```bash
kubectl apply -f my-mongodb.yaml
```

### Step 3: Monitor Deployment Progress

```bash
# Watch the instance status
kubectl get mongodbinstance my-first-mongodb -w

# Check detailed status
kubectl describe mongodbinstance my-first-mongodb

# View operator logs
kubectl logs -n mongodb-operator-system deployment/mongodb-operator-controller-manager -f
```

Deployment typically takes 5-10 minutes. You'll see the phase transition:
```
Pending → Creating → Running
```

### Step 4: Connect to MongoDB

Retrieve connection information:

```bash
# Get connection details
kubectl get mongodbinstance my-first-mongodb -o yaml

# Extract connection secret
export MONGODB_HOST=$(kubectl get mongodbinstance my-first-mongodb -o jsonpath='{.status.connection.hostname}')
export MONGODB_PORT=$(kubectl get mongodbinstance my-first-mongodb -o jsonpath='{.status.connection.port}')
export SECRET_NAME=$(kubectl get mongodbinstance my-first-mongodb -o jsonpath='{.status.connection.credentialsSecretRef}')

# Get credentials
export MONGODB_USER=$(kubectl get secret $SECRET_NAME -o jsonpath='{.data.username}' | base64 -d)
export MONGODB_PASS=$(kubectl get secret $SECRET_NAME -o jsonpath='{.data.password}' | base64 -d)

echo "Connection String: mongodb://$MONGODB_USER:$MONGODB_PASS@$MONGODB_HOST:$MONGODB_PORT"
```

### Step 5: Test Connection

From within the cluster:

```bash
# Run a MongoDB client pod
kubectl run -it --rm mongo-client --image=mongo:7.0 --restart=Never -- \
  mongosh "mongodb://$MONGODB_USER:$MONGODB_PASS@$MONGODB_HOST:$MONGODB_PORT"
```

From outside the cluster (requires LoadBalancer or port-forward):

```bash
# Port forward for local access
kubectl port-forward service/my-first-mongodb 27017:27017

# Connect with local mongosh client
mongosh "mongodb://$MONGODB_USER:$MONGODB_PASS@localhost:27017"
```

## Common Operations

### Scaling Resources

#### Increase Storage

```bash
kubectl patch mongodbinstance my-first-mongodb --type='json' -p='[
  {"op": "replace", "path": "/spec/storage/size", "value": "20Gi"}
]'
```

**Note**: Storage can only be increased, not decreased. The operation may take 10-30 minutes depending on data size.

#### Increase Memory and CPU

```yaml
# Edit the resource
kubectl edit mongodbinstance my-first-mongodb

# Update the resources section:
spec:
  resources:
    requests:
      memory: "4Gi"
      cpu: "2000m"
    limits:
      memory: "8Gi"
      cpu: "4000m"
```

**Note**: This triggers a pod restart, causing ~30-60 seconds of downtime.

### Configure Backups

#### Enable Automated Backups

First, create a secret with S3 credentials:

```yaml
# s3-credentials.yaml
apiVersion: v1
kind: Secret
metadata:
  name: s3-credentials
  namespace: default
type: Opaque
stringData:
  accessKeyId: "YOUR_AWS_ACCESS_KEY"
  secretAccessKey: "YOUR_AWS_SECRET_KEY"
  region: "us-east-1"
```

Apply:
```bash
kubectl apply -f s3-credentials.yaml
```

Update MongoDB instance to enable backups:

```bash
kubectl patch mongodbinstance my-first-mongodb --type='merge' -p='
spec:
  backup:
    enabled: true
    schedule: "0 2 * * *"
    retention: 14
    destination:
      type: s3
      bucket: "my-mongodb-backups"
      secretRef: "s3-credentials"
'
```

Backups will run daily at 2 AM UTC and retain 14 days of backups.

#### Manual Backup

Create a file named `manual-backup.yaml`:

```yaml
apiVersion: database.vteam.io/v1alpha1
kind: MongoDBBackup
metadata:
  name: my-first-mongodb-manual-backup
  namespace: default
spec:
  instanceRef:
    name: my-first-mongodb
  destination:
    type: s3
    bucket: "my-mongodb-backups"
    path: "manual-backups"
    secretRef: "s3-credentials"
```

Apply:
```bash
kubectl apply -f manual-backup.yaml

# Monitor backup progress
kubectl get mongodbbackup my-first-mongodb-manual-backup -w
```

#### Restore from Backup

Create a file named `restore.yaml`:

```yaml
apiVersion: database.vteam.io/v1alpha1
kind: MongoDBRestore
metadata:
  name: restore-from-manual-backup
  namespace: default
spec:
  targetInstanceRef:
    name: my-first-mongodb
  backupRef:
    name: my-first-mongodb-manual-backup
```

Apply:
```bash
kubectl apply -f restore.yaml

# Monitor restore progress
kubectl get mongodbrestore restore-from-manual-backup -w
```

**Warning**: Restore operations will replace existing data in the target instance.

### Health Monitoring

#### Check Instance Health

```bash
# Quick health check
kubectl get mongodbinstance my-first-mongodb -o jsonpath='{.status.health.status}'

# Detailed health information
kubectl get mongodbinstance my-first-mongodb -o jsonpath='{.status.health}' | jq .

# View all instances with health status
kubectl get mongodbinstance -o custom-columns=\
NAME:.metadata.name,\
PHASE:.status.phase,\
HEALTH:.status.health.status,\
AGE:.metadata.creationTimestamp
```

#### Troubleshoot Unhealthy Instances

```bash
# Check conditions
kubectl get mongodbinstance my-first-mongodb -o jsonpath='{.status.conditions}' | jq .

# View recent events
kubectl get events --field-selector involvedObject.name=my-first-mongodb

# Check pod logs
kubectl logs -l app.kubernetes.io/instance=my-first-mongodb

# Describe the StatefulSet
kubectl describe statefulset my-first-mongodb
```

### Delete MongoDB Instance

```bash
# Delete the instance
kubectl delete mongodbinstance my-first-mongodb

# Verify deletion
kubectl get mongodbinstance
```

**Important**: The operator will:
1. Check for recent backups (warning if none exist)
2. Delete StatefulSet (stops MongoDB pod)
3. Delete Service
4. Delete Secrets and ConfigMap
5. Optionally delete PVC (data will be lost!)

To preserve data, backup before deletion or manually keep the PVC:
```bash
# Remove owner reference from PVC before deleting instance
kubectl patch pvc my-first-mongodb-data --type=json -p='[{"op": "remove", "path": "/metadata/ownerReferences"}]'
```

## Production Deployment Example

For production workloads, use the following configuration:

```yaml
apiVersion: database.vteam.io/v1alpha1
kind: MongoDBInstance
metadata:
  name: production-mongodb
  namespace: production
spec:
  version: "7.0"

  # Storage
  storage:
    size: "100Gi"
    storageClassName: "gp3"  # AWS EKS example

  # Resources (adjust based on workload)
  resources:
    requests:
      memory: "8Gi"
      cpu: "4000m"
    limits:
      memory: "16Gi"
      cpu: "8000m"

  # Automated backups
  backup:
    enabled: true
    schedule: "0 2 * * *"
    retention: 30
    destination:
      type: s3
      bucket: "production-mongodb-backups"
      path: "production/production-mongodb"
      secretRef: "s3-credentials"

  # Network (internal only)
  network:
    serviceType: ClusterIP
    port: 27017
```

Apply with:
```bash
# Create namespace
kubectl create namespace production

# Create backup credentials secret
kubectl apply -f s3-credentials.yaml -n production

# Deploy MongoDB
kubectl apply -f production-mongodb.yaml

# Monitor deployment
kubectl get mongodbinstance -n production -w
```

## Configuration Reference

### Storage Classes by Platform

| Platform | Storage Class | Performance | Use Case |
|----------|--------------|-------------|----------|
| **AWS EKS** | `gp3` | 3000 IOPS, 125 MB/s | General purpose |
| **AWS EKS** | `io2` | Up to 64,000 IOPS | High performance |
| **Azure AKS** | `managed-premium` | 5000 IOPS, 200 MB/s | Production |
| **GCP GKE** | `pd-ssd` | 30 IOPS/GB | Standard |
| **GCP GKE** | `pd-extreme` | 120,000 IOPS | High performance |
| **OpenShift** | `rook-ceph-block` | Cluster-dependent | General purpose |

### Resource Sizing Guidelines

| Workload Type | Memory | CPU | Storage |
|---------------|--------|-----|---------|
| **Development** | 1-2Gi | 500m-1000m | 10-20Gi |
| **Testing** | 2-4Gi | 1000m-2000m | 20-50Gi |
| **Production (small)** | 4-8Gi | 2000m-4000m | 50-100Gi |
| **Production (medium)** | 8-16Gi | 4000m-8000m | 100-500Gi |
| **Production (large)** | 16-32Gi | 8000m-16000m | 500Gi-2Ti |

**MongoDB Memory Recommendation**: Allocate enough memory for:
- Working set (frequently accessed data)
- Internal cache (50% of RAM by default)
- Connections (1MB per connection)

### Backup Destination Configuration

#### S3 (AWS)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: s3-credentials
type: Opaque
stringData:
  accessKeyId: "AKIAIOSFODNN7EXAMPLE"
  secretAccessKey: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
  region: "us-east-1"
  # Optional: use S3-compatible endpoint
  # endpoint: "https://s3.amazonaws.com"
```

#### GCS (Google Cloud Storage)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gcs-credentials
type: Opaque
stringData:
  serviceAccountKey: |
    {
      "type": "service_account",
      "project_id": "your-project",
      "private_key_id": "key-id",
      "private_key": "-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n",
      "client_email": "mongodb-backups@your-project.iam.gserviceaccount.com",
      "client_id": "123456789",
      "auth_uri": "https://accounts.google.com/o/oauth2/auth",
      "token_uri": "https://oauth2.googleapis.com/token",
      "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
      "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/..."
    }
```

#### Azure Blob Storage

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: azure-credentials
type: Opaque
stringData:
  storageAccountName: "mystorageaccount"
  storageAccountKey: "abc123...xyz789=="
  # Or use SAS token
  # sasToken: "?sv=2021-06-08&ss=b&srt=sco&sp=rwdlac..."
```

## Troubleshooting

### Instance Stuck in "Creating" Phase

```bash
# Check operator logs
kubectl logs -n mongodb-operator-system deployment/mongodb-operator-controller-manager

# Check if PVC is bound
kubectl get pvc -l app.kubernetes.io/instance=my-first-mongodb

# Check StatefulSet status
kubectl describe statefulset my-first-mongodb

# Check pod status
kubectl get pods -l app.kubernetes.io/instance=my-first-mongodb
kubectl describe pod my-first-mongodb-0
```

**Common causes**:
- No storage available (check node capacity)
- Storage class doesn't exist
- Insufficient resources (memory/CPU)
- Image pull failures

### Health Status "Unavailable"

```bash
# Check MongoDB pod logs
kubectl logs my-first-mongodb-0

# Check service endpoints
kubectl get endpoints my-first-mongodb

# Try manual connection
kubectl exec -it my-first-mongodb-0 -- mongosh --eval "db.adminCommand('ping')"
```

**Common causes**:
- MongoDB container not ready
- Authentication issues
- Network policy blocking connections
- Resource constraints (OOMKilled)

### Backup Failures

```bash
# Check backup status
kubectl describe mongodbbackup <backup-name>

# Check backup job logs
kubectl get jobs -l backup-name=<backup-name>
kubectl logs job/<backup-job-name>
```

**Common causes**:
- Invalid storage credentials
- Network connectivity to storage
- Insufficient permissions on bucket
- Disk space on backup pod

### Restore Failures

```bash
# Check restore status
kubectl describe mongodbrestore <restore-name>

# Check restore job logs
kubectl get jobs -l restore-name=<restore-name>
kubectl logs job/<restore-job-name>
```

**Common causes**:
- Target instance not in Running phase
- Backup not found or corrupted
- Insufficient storage space
- MongoDB version mismatch

## Next Steps

- **Production Hardening**: Configure NetworkPolicies, ResourceQuotas, and PodDisruptionBudgets
- **Monitoring**: Integrate with Prometheus for metrics collection
- **Alerting**: Set up alerts for health status changes
- **Security**: Enable TLS for MongoDB connections (v2 feature)
- **High Availability**: Deploy MongoDB replica sets (v2 feature)

## Getting Help

- **Documentation**: https://github.com/vteam/mongodb-operator/docs
- **Issues**: https://github.com/vteam/mongodb-operator/issues
- **Discussions**: https://github.com/vteam/mongodb-operator/discussions
- **Slack**: #mongodb-operator on Kubernetes Slack

## Version Compatibility

| Operator Version | Kubernetes Version | MongoDB Version |
|------------------|-------------------|-----------------|
| v1.0.x | 1.24+ | 6.0, 7.0 |

**Note**: This is an MVP release. Future versions will support MongoDB replica sets, sharded clusters, and additional MongoDB versions.
