# Standalone NFS Server
We can expose a directory tree from a node to act as a nfs server.

```bash
sudo apt-get install nfs-kernel-server -y
```

Now edit the `exports`

```bash
# file: /etc/exports
# Use * for everyone or just the internal IPs
# /mnt/unionfs/storage *(rw,fsid=1,no_root_squash,insecure,async,no_subtree_check,anonuid=1000,anongid=1000)
/mnt/unionfs/storage 10.244.0.0/16(rw,fsid=1,no_root_squash,insecure,async,no_subtree_check,anonuid=1000,anongid=1000)
```

Start the nfs server

```bash
sudo exportfs -ra
```

# Deployment of PVCs via NFS
In order to serve persistent volumes via NFS (and therefore RWX) we can use the `nfs-server-provisioner` via helm.

**NB:** This assumes you have a `StorageClass` already setup called `local-path` (e.g. via the [local-path-provisioner](https://github.com/rancher/local-path-provisioner/blob/master/README.md)). The default k3s installation will deploy this for you automatically.

```bash
helm install nfs-server \
  stable/nfs-server-provisioner \
  --set persistence.enabled=true,persistence.storageClass=local-path,persistence.size=200Gi
```