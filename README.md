# Velero-longhorn-backup

# ➖➖ 🌟 Let's back up a Longhorn volume with Velero 🌟 ➖➖


---
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
      -**Velero uses the Velero-plugin-for-CSI to create a VolumeSnapshot resource.** \
      -**Longhorn recognizes the request and creates a volume snapshot.** \
      -**Longhorn, if properly configured, stores the snapshot in an S3 bucket.** \

---
**The installation of the MinIO on Ubuntu is explained here:** https://github.com/aleksandarkekic/Install-MiniO-on-Ubuntu

---
**Here is the configuration of Longhorn to use MinIO S3 for backups:** https://www.civo.com/learn/backup-longhorn-volumes-to-a-minio-s3-bucket

# Snapshot Controller and CSI Snapshot CRDs \
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
To make Velero create the VolumeSnapshots with our VolumeSnapshotClass we need the label **velero.io/csi-volumesnapshot-class: "true".**
Under parameters, we specify**type: bak**, which tells Longhorn that we want to make a Longhorn backup. An alternative would be type: snap for Longhorn snapshots (incremental backups, not to be confused with VolumeSnapshots). However, these do not yet have CSI support, so cannot be used here \

---
# Velero
```bash
# Install Velero CLI
wget https://github.com/vmware-tanzu/velero/releases/download/v1.11.0/velero-v1.11.0-linux-amd64.tar.gz
tar xvf velero-v1.11.0-linux-amd64.tar.gz
mv velero-v1.11.0-linux-amd64/velero /usr/local/bin

# Install Velero Server
cat << EOF >> credential-velero
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
EOF

velero install --provider velero.io/aws \
 --bucket velero --image velero/velero:v1.11.0 \
 --plugins velero/velero-plugin-for-aws:v1.7.0,velero/velero-plugin-for-csi:v0.4.0 \
 --backup-location-config region=minio-default,s3ForcePathStyle="true",s3Url=http://minio.minio:9000 \
 --features=EnableCSI --snapshot-location-config region=minio-default \
 --use-volume-snapshots=true --secret-file=./credential-velero


