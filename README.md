nfs-kapsule
===========

Tutorial to start a NFS Server in Kapsule with a Block PVC and expose it as a RWM PVC.

Target architecture
-------------------

The goal is to start a Pod (as a single replica Deployment to restart it in case of failure) with a Block PVC exported via a NFS Server.

The associated K8S Service is then used to create a Persistent Volume associated to a RWM Persistent Volume Claim.

This PVC can then be used as needed in other pods. Please remember that PV are NOT namespaced.

Used image
----------

The Dockerfile for the image used is in the folder `image`. Both NFSv3 and NFSv4 are used in this image to allow fallback.

The NFS services started are:
- rpcbind
- rpc.mountd
- rpc.nfsd
- rpc.statd

Since we will most probably don't use ID Mapping, rcp.idmapd is not started.

It uses a regular Alpine Linux image which does not support NFSv2 by default in its latest version (`-N 2` flag is not needed and would throw an error if added).

The port for rpc.mountd is fixed to 20048 to help build a K8S Service afterward.

This image MUST be run privileged to mount `/proc/fs/nfsd` and will crash if no volume is associated to `/exports` since overlayfs is not compatible with NFS.

Tricks
------

The Service is using a fixed ClusterIP:
- because the Service name is not resolvable by Kubelet on the host level.
- to avoid needing modification of the PV later on.

When creating such a NFS server, a specific free ClusterIP should be choosen in Kapsule Service CIDR: 10.32.0.0/12 (here `10.32.100.1`)

```
apiVersion: v1
kind: Service
metadata:
  name: nfs-service
spec:
  clusterIP: 10.32.100.1
  ports:
  - name: rpcbind
    port: 111
  - name: nfs
    port: 2049
  - name: mountd
    port: 20048
  selector:
    app: nfs-server
```

An initContainer is used in the NFS server Deployment to create the folder associated to the PV created. 

It changes the folder UID/GID to the target user in the client pod(s). (In this example `101` for `nginx` user in `nginx:alpine`).

```
initContainers:
- name: vol-init
  image: busybox
  command:
  - /bin/sh
  - -c
  - "mkdir -p /exports/pvc && chown -R 101:101 /exports/pvc"
  volumeMounts:
  - name: storage
    mountPath: /exports
```

When creating the PV, the same ClusterIP declared in the Service is used. The path is the folder created.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels:
    app: nfs-data
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: "10.32.100.1"
    path: "/pvc"
```

When using the PVC, in the client pod, make sure to change the fsGroup of the volume using the previous GID:

```
spec:
  securityContext:
    fsGroup: 101
    fsGroupChangePolicy: "Always"
```

Example usage
-------------

You will find a full manifest in the repository demonstrating the full procedure.

The example will:
- Create a Block PVC of 10GB,
- Expose this Block PVC using an NFS Server,
- Create the RWM NFS PVC using the associated Service,
- Start a 3 replicas Deployment of `nginx:alpine` and the `index.html` file is filed with the hostname of each pod via initContainer, proving the RWM access.
- Expose this Deployment using a Service.

Beware that this example, provided in `nfs-example.yml` is, on top of using a Block storage PVC, deploying a LoadBalancer type Service which will both lead to additional costs.

Pseudo storage class
--------------------

A workaround like https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner could be used to create sub-folder in the NFS exported folder to automatically create PVC instead of creating a PV + PVC manually.

The chart must then be applied after creating the NFS Server and associated Service and the ClusterIP provided in this TL;DR:

```
$ kubectl apply -f nfs-server-only.yml
$ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
$ helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=10.32.100.1 \
    --set nfs.path=/pvc
```

This does not satisfy full isolation of the data but may help some use-cases.

Please check also the following known pitfalls:
* The provisioned storage is not guaranteed. You may allocate more than the NFS share's total size. The share may also not have enough storage space left to actually accommodate the request.
* The provisioned storage limit is not enforced. The application can expand to use all the available storage regardless of the provisioned size.
* Storage resize/expansion operations are not presently supported in any form. You will end up in an error state: `Ignoring the PVC: didn't find a plugin capable of expanding the volume; waiting for an external controller to process this PVC.`

Availability
------------

This setup is *NOT* Highly Available. If the NFS server pod or its underlaying node fails, the RWM PVC is not available anymore and the associated app fails. But this mechanism does autoheal and survived both a reboot and a replace of the node hosting the NFS server pod.

To accelerate failure detection and recovery, configure liveness probes.

For a HA RWM volume, you can switch to CephFS using RookFS.
