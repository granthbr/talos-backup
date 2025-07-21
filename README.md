# talos-backup

A reliable backup tool for Talos Linux-based Kubernetes clusters that creates encrypted etcd snapshots and stores them in S3-compatible storage.

## Overview

talos-backup runs as a Kubernetes CronJob that:
1. Creates etcd snapshots from your Talos cluster
2. Optionally compresses snapshots using zstd
3. Encrypts snapshots using age encryption
4. Uploads encrypted snapshots to S3-compatible storage

## Prerequisites

### 1. Talos Configuration

Enable Kubernetes API access to Talos in your machine configuration:

```yaml
spec:
  machine:
    features:
      kubernetesTalosAPIAccess:
        enabled: true
        allowedRoles:
        - os:etcd:backup
        allowedKubernetesNamespaces:
        - default  # or your preferred namespace
```

Apply this configuration to your Talos nodes and reboot if necessary.

### 2. Age Encryption Keys

Generate an age key pair for encryption:

```bash
# Install age if not already installed
# On most systems: apt install age, brew install age, etc.

# Generate key pair
age-keygen -o backup-key.txt

# The output will show:
# Public key: age1khpnnl86pzx96ttyjmldptsl5yn2v9jgmmzcjcufvk00ttkph9zs0ytgec
# Generated new private key to "backup-key.txt"
```

**Important**: Store the private key (`backup-key.txt`) securely - you'll need it to decrypt backups. The public key goes in your Kubernetes configuration.

## Installation

### Basic S3 Setup (AWS)

For standard AWS S3, customize the `cronjob.sample.yaml` file:

```yaml
env:
  - name: AWS_ACCESS_KEY_ID
    value: "your-access-key-id"
  - name: AWS_SECRET_ACCESS_KEY
    value: "your-secret-access-key"  # Consider using a Kubernetes secret
  - name: AWS_REGION
    value: "us-west-2"
  - name: BUCKET
    value: "my-talos-backups"
  - name: CLUSTER_NAME
    value: "prod-cluster"
  - name: S3_PREFIX
    value: "cluster-backups"  # Optional: organizes backups in subdirectory
  - name: AGE_X25519_PUBLIC_KEY
    value: "age1khpnnl86pzx96ttyjmldptsl5yn2v9jgmmzcjcufvk00ttkph9zs0ytgec"
  - name: ENABLE_COMPRESSION
    value: "true"  # Recommended for smaller backup files
```

### Minio S3 Setup

For Minio (including in-cluster Minio), configure these additional environment variables:

```yaml
env:
  - name: AWS_ACCESS_KEY_ID
    value: "minio-access-key"
  - name: AWS_SECRET_ACCESS_KEY
    value: "minio-secret-key"
  - name: AWS_REGION
    value: "us-east-1"  # Minio default, can be any value
  - name: CUSTOM_S3_ENDPOINT
    value: "https://minio.my-cluster.local:9000"  # Your Minio endpoint
  - name: USE_PATH_STYLE
    value: "true"  # Required for Minio
  - name: BUCKET
    value: "talos-backups"
  - name: CLUSTER_NAME
    value: "my-cluster"
  - name: AGE_X25519_PUBLIC_KEY
    value: "age1khpnnl86pzx96ttyjmldptsl5yn2v9jgmmzcjcufvk00ttkph9zs0ytgec"
  - name: ENABLE_COMPRESSION
    value: "true"
```

**Important for in-cluster Minio**: If your Minio is running in the same cluster you're backing up, consider this creates a single point of failure. For production, use external S3 storage.

### External S3-Compatible Services

For other S3-compatible services (DigitalOcean Spaces, Backblaze B2, etc.):

```yaml
env:
  - name: AWS_ACCESS_KEY_ID
    value: "your-service-key"
  - name: AWS_SECRET_ACCESS_KEY
    value: "your-service-secret"
  - name: AWS_REGION
    value: "nyc3"  # Service-specific region
  - name: CUSTOM_S3_ENDPOINT
    value: "https://nyc3.digitaloceanspaces.com"
  - name: USE_PATH_STYLE
    value: "false"  # Most services use virtual-hosted style
  - name: BUCKET
    value: "my-backup-space"
  - name: CLUSTER_NAME
    value: "production"
  - name: AGE_X25519_PUBLIC_KEY
    value: "age1khpnnl86pzx96ttyjmldptsl5yn2v9jgmmzcjcufvk00ttkph9zs0ytgec"
```

## Deployment

1. **Customize the configuration**:
   ```bash
   cp cronjob.sample.yaml cronjob.yaml
   # Edit cronjob.yaml with your configuration
   ```

2. **Create the bucket** (if it doesn't exist):
   ```bash
   # For AWS CLI
   aws s3 mb s3://my-talos-backups --region us-west-2
   
   # For Minio Client (mc)
   mc mb myminio/talos-backups
   ```

3. **Deploy the CronJob**:
   ```bash
   kubectl apply -f cronjob.yaml
   ```

4. **Test the backup**:
   ```bash
   # Create a test job from the cronjob
   kubectl create job --from=cronjob/talos-backup backup-test
   
   # Check the job status
   kubectl get jobs
   kubectl logs job/backup-test
   ```

## Configuration Reference

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `AWS_ACCESS_KEY_ID` | Yes | - | S3 access key |
| `AWS_SECRET_ACCESS_KEY` | Yes | - | S3 secret key |
| `AWS_REGION` | Yes | - | S3 region |
| `BUCKET` | Yes | - | S3 bucket name |
| `AGE_X25519_PUBLIC_KEY` | Yes | - | Age public key for encryption |
| `CUSTOM_S3_ENDPOINT` | No | AWS default | Custom S3 endpoint URL |
| `USE_PATH_STYLE` | No | `false` | Use path-style URLs (required for Minio) |
| `CLUSTER_NAME` | No | Context name | Cluster identifier for backups |
| `S3_PREFIX` | No | Cluster name | S3 key prefix for organization |
| `ENABLE_COMPRESSION` | No | `false` | Compress snapshots with zstd |
| `DISABLE_ENCRYPTION` | No | `false` | Skip encryption (not recommended) |

### Schedule Configuration

The default schedule is every 10 minutes (`0/10 * * * *`). Common alternatives:

```yaml
# Every hour at minute 0
schedule: '0 * * * *'

# Every 6 hours
schedule: '0 */6 * * *'

# Daily at 2 AM
schedule: '0 2 * * *'

# Every 15 minutes
schedule: '*/15 * * * *'
```

## Backup Management

### Backup File Structure

Backups are stored with this naming pattern:
```
s3://bucket/prefix/cluster-name-YYYY-MM-DD-HH-MM-SS.snapshot.age
```

Example:
```
s3://talos-backups/prod-cluster/prod-cluster-2024-01-15-14-30-00.snapshot.age
```

### Restoring from Backup

1. **Download the backup**:
   ```bash
   aws s3 cp s3://talos-backups/prod-cluster/prod-cluster-2024-01-15-14-30-00.snapshot.age ./backup.snapshot.age
   ```

2. **Decrypt the backup**:
   ```bash
   age -d -i backup-key.txt backup.snapshot.age > etcd-snapshot.db
   ```

3. **Use with Talos recovery**:
   ```bash
   talosctl bootstrap --recover-from=./etcd-snapshot.db
   ```

### Monitoring Backups

Check backup job status:
```bash
# View recent jobs
kubectl get jobs -l job-name=talos-backup

# View cronjob status
kubectl get cronjobs

# Check logs
kubectl logs -l job-name=talos-backup --tail=100
```

Set up monitoring alerts for failed backup jobs in your monitoring system.

## Security Considerations

1. **Encryption**: Always use age encryption in production
2. **Secrets**: Store AWS credentials as Kubernetes secrets instead of plain environment variables
3. **Access**: Use IAM roles with minimal required permissions
4. **Key Management**: Store the age private key securely and separately from your cluster
5. **Network**: For in-cluster Minio, ensure backups also go to external storage

### Using Kubernetes Secrets for Credentials

Create a secret for S3 credentials:

```bash
kubectl create secret generic s3-credentials \
  --from-literal=AWS_ACCESS_KEY_ID=your-key \
  --from-literal=AWS_SECRET_ACCESS_KEY=your-secret
```

Reference in your CronJob:
```yaml
env:
  - name: AWS_ACCESS_KEY_ID
    valueFrom:
      secretKeyRef:
        name: s3-credentials
        key: AWS_ACCESS_KEY_ID
  - name: AWS_SECRET_ACCESS_KEY
    valueFrom:
      secretKeyRef:
        name: s3-credentials
        key: AWS_SECRET_ACCESS_KEY
```

## Troubleshooting

### Common Issues

1. **"Failed to load AWS configuration"**
   - Check AWS credentials are set correctly
   - Verify the bucket exists and is accessible

2. **"Failed to create talos client"**
   - Ensure `kubernetesTalosAPIAccess` is enabled in machine config
   - Verify the namespace is allowed in `allowedKubernetesNamespaces`

3. **"Access Denied" with Minio**
   - Set `USE_PATH_STYLE=true` for Minio
   - Check Minio credentials and bucket permissions

4. **Job fails silently**
   - Check pod logs: `kubectl logs -l job-name=talos-backup`
   - Verify age public key format

### Debug Mode

To test configuration without creating a full job:

```bash
# Run a one-time pod
kubectl run talos-backup-debug --image=your-registry/talos-backup:latest \
  --env="AWS_ACCESS_KEY_ID=..." \
  --env="AWS_SECRET_ACCESS_KEY=..." \
  # ... other env vars
  --rm -it --restart=Never -- /talos-backup
```

## Development

Build the binary locally:
```bash
make talos-backup
```

Build container image:
```bash
make REGISTRY=registry.example.com USERNAME=myusername PUSH=true TAG=latest image-talos-backup
```

## Contributing

This project uses the Mozilla Public License 2.0. See [LICENSE](LICENSE) for details.