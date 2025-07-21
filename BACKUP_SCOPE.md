# Backup Scope and Contents

This document details exactly what is and isn't backed up by the talos-backup tool.

## What is Backed Up

**Primary Data:**
- **etcd snapshot** - This is the core backup content. The tool creates a complete snapshot of the etcd database.

## What etcd Contains (What You're Actually Backing Up)

The etcd database in a Kubernetes cluster contains:

**Kubernetes Resources:**
- All Kubernetes API objects (Deployments, Services, ConfigMaps, Secrets, etc.)
- Namespace definitions
- RBAC policies (Roles, RoleBindings, ServiceAccounts)
- Custom Resource Definitions (CRDs) and custom resources
- Ingress configurations
- PersistentVolume and PersistentVolumeClaim definitions
- Network policies
- Cluster-wide configuration

**Talos-Specific Data:**
- Talos machine configurations
- Cluster membership information
- Bootstrap tokens and certificates

## What is NOT Backed Up

**Application Data:**
- Data stored in PersistentVolumes (your actual application databases, file storage, etc.)
- Container images
- Logs
- Metrics/monitoring data stored outside etcd
- Any data in mounted volumes or hostPath mounts

**Infrastructure:**
- The Talos OS installation itself
- Container runtime state
- Node-specific configurations not stored in etcd

## Backup Process Flow

The backup process:

1. **Takes etcd snapshot** using Talos API
2. **Optionally compresses** with zstd algorithm
3. **Encrypts** with age encryption (unless disabled)
4. **Uploads to S3** with naming pattern: `cluster-name-YYYY-MM-DDTHH:MM:SSZ.snap.age`

## What This Means for Recovery

**You CAN restore:**
- Complete Kubernetes cluster state
- All deployed applications and their configurations
- Secrets, ConfigMaps, and other Kubernetes resources
- Talos cluster configuration

**You CANNOT restore:**
- Application data (databases, user files, etc.) - these need separate backup strategies
- Container images (will be re-pulled from registries)
- Any data stored outside of etcd

## Complete Backup Strategy Recommendations

For a complete backup strategy, you should also:

1. **Backup persistent volumes** separately using tools like:
   - Velero with volume snapshots
   - Storage-specific backup tools (AWS EBS snapshots, etc.)
   - Application-specific backup tools

2. **Backup container registry** or ensure external registry availability:
   - Private registry backups
   - Documented image versions and sources
   - Image vulnerability scanning and compliance records

3. **Backup application-specific data**:
   - Database dumps or replicas
   - File storage backups
   - Configuration files not stored in ConfigMaps

4. **Document external dependencies**:
   - DNS configurations
   - Load balancer settings
   - SSL certificates and PKI
   - Network configurations
   - External service integrations

## Key Insight

The etcd backup is essentially the "brain" of your cluster - it knows what should be running and how it should be configured, but not the actual data your applications have created. Think of it as backing up the blueprint of your cluster, not the contents of the rooms.

## Recovery Scenarios

**Scenario 1: Complete cluster loss**
- etcd backup restores cluster configuration
- Applications will restart but with empty data stores
- Requires separate data restoration

**Scenario 2: Partial corruption**
- etcd backup can restore cluster state to a point in time
- Running applications may need to be restarted
- Data integrity depends on separate backup strategies

**Scenario 3: Configuration rollback**
- etcd backup allows reverting cluster configuration changes
- Useful for rolling back problematic deployments or configuration changes
- Application data remains intact