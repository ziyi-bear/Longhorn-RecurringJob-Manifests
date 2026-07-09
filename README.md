# Longhorn RecurringJob Manifests

This repository contains Kubernetes manifests for Longhorn `RecurringJob` custom resources. It is designed to be automatically synced to Kubernetes clusters using **Rancher Fleet** for GitOps-based configuration management.

The configurations are optimized for **low-end clusters** to minimize resource usage (CPU/IOPS) by serializing jobs and minimizing the local storage footprint.

---

## Included Recurring Jobs

This repository provides two recurring jobs targeted at the `longhorn-system` namespace:

1. **Local Minimal Snapshot** ([snapshot-local-minimal.yaml](file:///d:/Anti-github/Longhorn-RecurringJob-Manifests/snapshot-local-minimal.yaml))
   - **Task**: `snapshot`
   - **Schedule**: `0 1 * * *` (Daily at 1:00 AM)
   - **Retention**: Keeps only the `2` most recent snapshots locally.
   - **Concurrency**: `1` (Processes volumes one at a time to prevent high disk IOPS spikes).
   
2. **Yearly S3 Backup** ([backup-s3-yearly.yaml](file:///d:/Anti-github/Longhorn-RecurringJob-Manifests/backup-s3-yearly.yaml))
   - **Task**: `backup`
   - **Schedule**: `0 2 * * *` (Daily at 2:00 AM, running after the local snapshot)
   - **Retention**: Keeps `100` backups in AWS S3 or compatible object storage (the maximum allowed by Longhorn's webhook validator).
   - **Concurrency**: `1` (Processes one volume backup at a time to minimize CPU and outbound network load).

---

## Syncing with Rancher Fleet

To automatically sync these recurring jobs across your cluster(s) using Rancher Fleet, follow these steps:

### 1. Create a Fleet GitRepo Resource
Define a Fleet `GitRepo` manifest to track this repository and target your cluster(s).

```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: longhorn-recurring-jobs
  namespace: fleet-default
spec:
  repo: https://github.com/ziyi-bear/Longhorn-RecurringJob-Manifests.git
  branch: main
  targets:
    - clusterSelector:
        matchLabels:
          provider.cattle.io: rke2 # Change your Cluster Name
```

Apply this `GitRepo` in your Rancher management cluster or create it via the Rancher UI under **Continuous Delivery**.

### 2. Apply Jobs to Volumes

Once Rancher Fleet deploys these manifests, the `RecurringJob` custom resources will exist in your downstream clusters under the `longhorn-system` namespace. You must then associate them with your volumes.

#### Option A: Label StorageClasses (Recommended for GitOps)
To automatically apply these recurring jobs to all volumes provisioned by a specific `StorageClass`, add the `recurringJobSelector` parameter to your `StorageClass` manifest:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-recurring
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "2880"
  # Assign the recurring jobs from this repository:
  recurringJobSelector: '[
    {"name":"snapshot-local-minimal", "isGroup":false},
    {"name":"backup-s3-yearly", "isGroup":false}
  ]'
```

#### Option B: Label/Annotate Volumes Manually
You can assign these recurring jobs to individual volumes by editing the volume labels or using the Longhorn UI:
- Label name: `recurring-job.longhorn.io/job-name`
- Label value: `snapshot-local-minimal` or `backup-s3-yearly`

---

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](file:///d:/Anti-github/Longhorn-RecurringJob-Manifests/LICENSE) file for details.
