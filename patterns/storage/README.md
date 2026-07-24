# Pattern: storage split (example only)

| Workload need | Pattern |
|---------------|---------|
| VM disks / RWX-ish block | Replicated block (LINSTOR/DRBD-class) |
| Artifacts / models | S3-compatible object |
| Bulk backup | ZFS-class dataset + offsite copy (3-2-1) |

Example StorageClass name only (not a live cluster export):

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: replicated-block
provisioner: linstor.csi.linbit.com
allowVolumeExpansion: true
reclaimPolicy: Delete
```

Tune parameters per environment; do not copy production parameters from private catalogs into public git.
