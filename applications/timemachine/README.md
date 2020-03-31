Ideally, deploys a [docker-samba-timemachine](https://github.com/darth-veitcher/docker-samba-timemachine) with a persistent volume.

In reality I struggled with any of the smb based images as they kept failing with "unable to create backup disk" (despite finding the timemachine container and exposed volume). As a result this uses one of the older afp-based images for the moment [odarriba/docker-timemachine](https://github.com/odarriba/docker-timemachine).

The working docker command is:

```bash
docker run -d \
  -h timemachine \
  --name timemachine \
  --restart=unless-stopped \
  -v /mnt/local/media/timemachine:/timemachine \
  -it \
  -p 548:548 \
  -p 636:636 \
  --ulimit nofile=65536:65536 \
odarriba/timemachine
```

Now add a user and associated volume.

```bash
docker exec timemachine add-account USERNAME PASSWORD VOL_NAME VOL_ROOT [VOL_SIZE_MB]
```

* `VOL_NAME` will be the name of the volume shown on your OSX as the network drive
* `VOL_ROOT` should be an absolute path, preferably a sub-path of /timemachine (e.g., /timemachine/backup), so it will be stored in the according sub-path of your external volume.
* `VOL_SIZE_MB` is an optional parameter. It indicates the max volume size for that user.