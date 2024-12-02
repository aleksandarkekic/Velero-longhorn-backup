# Velero-longhorn-backup
===========================================Now we will see how to perform a backup of a Longhorn volume with Velero===========================================
For our purposes, Velero is practically without alternative. It can back up all K8s resources, but also volume data.
Backups can be executed automatically with schedules. It can be extended with plugins: For example, a S3 bucket can be connected. 
If, like us, you do not use the cluster-internal storage provider, but for example Longhorn, you can easily connect it with a plug-in for the container storage interface (CSI)
How Backup & Restore works

