# Setting up Longhorn for clustered storage.

Deploy with the manifest for the latest stable version.

**NB:** If you have nodes with differing capabilities and you'd like to manage which ones get the Longhorn deployments (which can be resource intensive) then it's easier to use the Helm approach below and specify. See [Setting up Node Selector for Longhorn](https://longhorn.io/docs/1.1.1/advanced-resources/deploy/node-selector/#setting-up-node-selector-for-longhorn).

=== "Manifest"
  ```zsh
  export VERSION=1.1.1
  kubectl apply -f "https://raw.githubusercontent.com/longhorn/longhorn/v${VERSION}/deploy/longhorn.yaml"
  ```

=== "Helm"
  We need to modify the following values:

  * longhornManager.nodeSelector
  * longhornUI.nodeSelector
  * longhornDriver.nodeSelector

  I'm going to label up the nodes I want to have the pods.

  ```zsh
  kubectl label node mini-server.jamesveitch.me storage=true
  ```

  I can now create a `values.yaml` or feed this directly to helm.

  ```zsh
  helm repo add longhorn https://charts.longhorn.io
  helm repo update
  kubectl create ns longhorn-system
  helm install longhorn longhorn/longhorn --namespace longhorn-system \
    --set 'longhornManager.nodeSelector.storage=true' \
    --set 'longhornUI.nodeSelector.storage=true'
  ```

## Setup access to the UI

We will secure (initially) the access to the admin panel with basic authentication.

### Create the secret

A decent way to generate a quick, usable, password is with `openssl rand 20 -hex`. Where we're specifying 20 digits in total and using hexadecimal encoding so that the characters are all printable.

```zsh
USER=admin; PASSWORD=cdf4aca2d91bf1db37ecf02973bc21cc52678417; echo "${USER}:$(openssl passwd -stdin -apr1 <<< ${PASSWORD})" >> auth
kubectl -n longhorn-system create secret generic basic-auth --from-file=auth
rm -f auth
```

### Setup access via Ingress

I've currently chosen not to expose this to the outside world for obvious reasons until I've implemented a VPN. To access it use kubectl to port forward to your localhost.

```zsh
kubectl port-forward service/longhorn-frontend -n longhorn-system 3000:80
```

This will allow you to browse the UI from [localhost:3000](http://127.0.0.1:3000)

## Create hyper-converged storage

As explained [in the docs](https://longhorn.io/docs/1.1.1/high-availability/data-locality/#set-the-data-locality-for-individual-volumes-using-a-storageclass) I'll adjust the default parameters for the longhorn storage class so it encourages data to be resident next to the containers running.

```zsh hl_lines="9 10"
cat <<EOF | kubectl apply -f -
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: hyper-converged
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "2"
  dataLocality: "best-effort"
  staleReplicaTimeout: "2880" # 48 hours in minutes
  fromBackup: ""
EOF
```

This will have two replicas of the data and attempt to maintain a copy on the same node as the pod. We now should have the following classes available.

```zsh
➜ kubectl get storageclass

NAME                          PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
microk8s-hostpath (default)   microk8s.io/hostpath   Delete          Immediate           false                  11h
longhorn                      driver.longhorn.io     Delete          Immediate           true                   10h
hyper-converged               driver.longhorn.io     Delete          Immediate           true                   3s
```

I'd like to set the new `hyper-converged` class as the [default](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/) for the cluster as opposed to the current `microk8s-hostpath`.

Mark `microk8s-hostpath` as non-default.

```zsh
kubectl patch storageclass microk8s-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

Mark `hyper-converged` as the new default.

```zsh
kubectl patch storageclass hyper-converged -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

We can now check this change.

```zsh
➜ kubectl get storageclass

NAME                        PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
longhorn                    driver.longhorn.io     Delete          Immediate           true                   10h
microk8s-hostpath           microk8s.io/hostpath   Delete          Immediate           false                  11h
hyper-converged (default)   driver.longhorn.io     Delete          Immediate           true                   3m37s
```

## Create a workload

To test it out let's create a basic workload that will use this storage. For the sake of simplicity I'll use our Minio workload as an example.

??? example "MinIO manifest"
    The sole difference between the below manifest and the original is we removed any reference to a preferred `storageClassName`.

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: minio-config
      labels:
        app: minio
    data:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: BucketMaster
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: minio-claim
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: minio
    spec:
      selector:
        matchLabels:
          app: minio
      replicas: 1
      template:
        metadata:
          labels:
            app: minio
        spec:
          containers:
            - name: minio
              image: minio/minio:latest
              imagePullPolicy: "IfNotPresent"
              args: ["server", "/data"]
              ports:
              - containerPort: 9000
              envFrom:
              - configMapRef:
                  name: minio-config
              volumeMounts:
              - name: data
                mountPath: /data
              # https://docs.min.io/docs/minio-monitoring-guide.html
              livenessProbe:
                httpGet:
                  path: /minio/health/live
                  port: 9000
                failureThreshold: 1
                periodSeconds: 10
          volumes:
          - name: data
            persistentVolumeClaim:
              claimName: minio-claim
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: minio
      labels:
        app: minio
    spec:
      selector:
        app: minio
      ports:
        - protocol: TCP
          name: http
          port: 9000
          targetPort: 9000
    ```

In theory, removing the specification around the `storageClassName` completely should make the cluster provision storage via the default (in this case our `hyper-converged` class).

```zsh hl_lines="6"
➜ kubectl describe po minio-65b4ffcbd7-g7bkj

Name:         minio-65b4ffcbd7-g7bkj
Namespace:    default
Priority:     0
Node:         node-1.jamesveitch.dev/XX.XX.XX.XX
Start Time:   Tue, 22 Jun 2021 21:40:45 +0100
```

So the pod has been scheduled on `node-1`.

```zsh
➜ kubectl get pvc        

NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
minio-claim   Bound    pvc-14bea17e-05f4-4714-b15c-c2c53024d0b1   1Gi        RWO            hyper-converged   117s
```

The pvc has been bound and provisioned with `hyper-converged`. Navigating to the frontend we can see there's a replica on `node-1` and a replica on `node-2` which is perfect.

### Test it ...

I'm going to upload some files to a bucket, then kill `node-1` and see what happens ...

**NB:** At the moment we don't have any sort of load balancing in place so, for now, I'll need to set one of the other nodes IP addresses in the `~/.kube/config`.

```zsh
➜ kubectl config use-context dev-node-2  

Switched to context "dev-node-2".
```

Now with two terminals open I'll kill `node-1` and watch the pod get rescheduled.

```zsh
➜ kubectl get po -w

NAME                     READY   STATUS             RESTARTS   AGE
minio-65b4ffcbd7-g7bkj   1/1     Running            0          19m
minio-65b4ffcbd7-g7bkj   1/1     Running            0          20m
minio-65b4ffcbd7-g7bkj   0/1     Unknown            0          22m
minio-65b4ffcbd7-g7bkj   0/1     Terminating        0          23m
minio-65b4ffcbd7-xjsfg   0/1     Pending            0          0s
minio-65b4ffcbd7-xjsfg   0/1     Pending            0          0s
minio-65b4ffcbd7-xjsfg   0/1     ContainerCreating  0          0s
minio-65b4ffcbd7-g7bkj   0/1     Terminating        0          23m
minio-65b4ffcbd7-g7bkj   0/1     Terminating        0          23m
minio-65b4ffcbd7-xjsfg   0/1     ContainerCreating  0          64s
minio-65b4ffcbd7-xjsfg   1/1     Running            0          77s
```

The pod gets rescheduled onto `node-3`.

```zsh hl_lines="6"
➜ kubectl describe po minio-65b4ffcbd7-xjsfg 

Name:         minio-65b4ffcbd7-xjsfg
Namespace:    default
Priority:     0
Node:         node-3.jamesveitch.dev/XX.XX.XX.XX
Start Time:   Tue, 22 Jun 2021 22:03:50 +0100
```

Logging into the web interface the two files I uploaded are still there. The pv has been attached and the underlying storage has now moved across to `node-3`.
