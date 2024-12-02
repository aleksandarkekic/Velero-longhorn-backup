# Velero-longhorn-backup
====================**Now we will see how to perform a backup of a Longhorn volume with Velero**====================

For our purposes, Velero is practically without alternative. It can back up all K8s resources, but also volume data.
Backups can be executed automatically with schedules. It can be extended with plugins: For example, a S3 bucket can be connected. 
If, like us, you do not use the cluster-internal storage provider, but for example Longhorn, you can easily connect it with a plug-in for the container storage interface (CSI)
How Backup & Restore works

Velero is a Kubernetes operator. An operator listens for and processes the creation, modification, and deletion of specific Kubernetes resources. For example, Velero listens for resources of type velero.io/v1/Backup and velero.io/v1/Restore
Backup
The velero backup create <backup-name> command creates a new backup resource with the specified name. The Velero server recognizes the created resource and starts the backup process,
which collects and stores all the resources specified in the backup. To store the backup outside the cluster, we can include an object store plugin. In our case, it is the Velero-plugin-for-AWS, which writes the data to a S3 bucket.

But what happens to the volume data?
If you use normal Kubernetes volumes, Velero writes them to the MinIO bucket along with the other resources. Since we are using Longhorn, this is not possible. 
However, Longhorn supports the container storage interface, through which Velero can tell Longhorn to create a backup. This is where the Velero-plugin-for-CSI comes in: 
it creates a VolumeSnapshot, which tells Longhorn to create a backup. 
Now, if Longhorn is configured correctly, it writes this backup to a S3 bucket.

Velero cannot directly copy data from Longhorn volumes to object storage. Instead: \
      -Velero uses the Velero-plugin-for-CSI to create a VolumeSnapshot resource. \
      -Longhorn recognizes the request and creates a volume snapshot. \
      -Longhorn, if properly configured, stores the snapshot in an S3 bucket. \


The installation of the MinIO service on Linux is explained here: https://github.com/aleksandarkekic/Install-MiniO-on-Ubuntu


Here is the configuration of Longhorn to use MinIO S3 for backups: https://www.civo.com/learn/backup-longhorn-volumes-to-a-minio-s3-bucket

**Snapshot Controller and CSI Snapshot CRDs** \
To create CSI snapshots, we need a snapshot controller and the CSI snapshot CRDs. Since these are not installed by default on K3s, we need to install them manually:
```bash
kubectl -n kube-system create -k "github.com/kubernetes-csi/external-snapshotter/client/config/crd?ref=release-5.0"
kubectl -n kube-system create -k "github.com/kubernetes-csi/external-snapshotter/deploy/kubernetes/snapshot-controller?ref=release-5.0"
```
In order for the snapshot controller to use Longhorn for the snapshots, we need to create a VolumeSnapshotClass:
```yaml
kind: VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
metadata:
  name: longhorn-snapshot-vsc
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: driver.longhorn.io
deletionPolicy: Delete
parameters:
  type: bak
```

