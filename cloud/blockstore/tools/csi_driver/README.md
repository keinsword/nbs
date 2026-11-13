# Compute-node storage publications

In VM mode, compute-node can call the CSI Node service directly. Kubelet still
sets up the launcher's ordinary hostPath mount, but does not issue CSI calls for
these attachments.

For a compute-node publication, use a volume ID of `<resourceID>#<instanceID>`
and set `VolumeContext.instanceId` to the same instance ID. Publish must supply
the launcher Pod UID in `VolumeContext["csi.storage.k8s.io/pod.uid"]`. The driver
uses that field for instance-qualified staged VM volumes. If it is absent, the
driver falls back to parsing the kubelet target path. Other volume flows retain
path-based Pod identification because their cleanup depends on that path.
A mismatched instance ID or a missing Pod UID that cannot be obtained from the
path returns `InvalidArgument`.

The node DaemonSet manifest exposes these compute-node defaults:

| Purpose | Host path and CSI container path |
|---|---|
| Persistent stage records | `/var/lib/computenode/storage/staging` |
| Publication bind mounts | `/var/lib/computenode/storage/published` |

Both mounts must use the same absolute host and container paths. If compute-node
uses different `staging_root` or `publication_root` values, change the matching
`hostPath.path` and `volumeMounts.mountPath` values in the DaemonSet. The publication
mount uses `Bidirectional` propagation so CSI bind mounts reach the host. The
host mount hierarchy must support propagation. Keep staging on persistent host
storage rather than the endpoint socket tmpfs.

Compute-node stages under `<staging_root>/<instanceID>/<resourceID>` and publishes
under `<publication_root>/<VMIUID>/<resourceID>`. Launcher admission must mount
`<publication_root>/<VMIUID>` at `/var/run/compute-storage` with `HostToContainer`
propagation. These attachments must not also be declared as Pod PVC/CSI volumes.

Stage starts the backend endpoint and saves its cleanup record. Publish binds the
endpoint directory to the requested target. For instance-qualified VM volumes,
Unpublish removes only the publication; Unstage stops the endpoint. Compute-node
must remove all local publications before it calls Unstage. This applies to NBS
disks and YDB FileStore filesystems exposed through virtio-fs, not the NFS protocol.
Backend socket permissions must allow the launcher's QEMU process to connect.

This repository contains a static node DaemonSet manifest. The production CSI
Helm templates and values are maintained in Nebo and need the same mounts.

# Volume expansion

In pod mode, online expansion is enabled by default. The driver advertises
node expansion only: `NodeExpandVolume` grows the NBS disk, refreshes the NBD
endpoint, and expands the filesystem.

To require offline expansion, pass `--offline-resize` to the
CSI driver in the controller Deployment and node DaemonSet. The driver
advertises `VolumeExpansion.OFFLINE` and rejects controller expansion with
`FailedPrecondition` while `StatVolume` reports clients. A failed `StatVolume`
also prevents expansion. Requests already satisfied by the disk's current
capacity reported by `StatVolume` succeed even with clients, without resizing
the disk again.

In offline mode, the driver advertises controller expansion only and returns
`NodeExpansionRequired: false`. `NodeStageVolume` creates the endpoint with the
new disk capacity, then checks and expands the filesystem after mounting it.
If filesystem expansion fails, staging fails and can be retried. Raw block
volumes only need the endpoint and device to be staged; no filesystem resize
or endpoint refresh is needed.

Keep `allowVolumeExpansion: true` in the StorageClass and the external-resizer's
`--handle-volume-inuse-error=true` (the default). To expand an in-use volume:

1. Increase the PVC's requested storage.
2. Stop all pods using it (for example, scale the workload to zero) and wait
   for the volume to be unstaged and its NBS clients disconnected.
3. Wait for controller expansion to complete (the PV reports the new capacity),
   then restore the workload. Filesystem expansion completes during staging.

Kubernetes does not stop or restart pods automatically for offline expansion.
The client check is not atomic with `ResizeVolume`; it does not prevent a
concurrent client connection outside this workflow.

The flag only applies to pod mode. VM mode keeps its external backend resize
workflow: node expansion refreshes the NBS endpoint, and NFS expansion remains
unchanged.

# Setup VSCode

./ya ide vscode-go -P workspace/csi-driver --no-gopls-fix cloud/blockstore/tools/csi_driver

# Install local k8s cluster (using Minikube)

\# Install kubectl

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```

\# Install minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo chmod +x minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

\# Install docker

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo usermod -aG docker $USER && newgrp docker
```

\# Start a local k8s cluster
\# Use --driver=docker to avoid blkUtil.AttachFileDevice failed error
\# https://github.com/kubernetes/minikube/issues/8284

```bash
minikube start --driver=docker
```

\# Check k8s cluster

```bash
minikube status
kubectl cluster-info
kubectl get nodes
```

\# Enable nbd devices
```bash
minikube ssh -- sudo modprobe nbd nbds_max=128
minikube ssh -- ls -l /dev/ | grep nbd
```
The last command should show the nbd devices, if not then try to restart minikube
```bash
minikube stop
minikube start
```

# Build csi-driver
```bash
./ya make -r cloud/blockstore/tools/csi_driver

```

# Replace symlink with binary

```bash
DRIVER_PATH=$(readlink -f "./cloud/blockstore/tools/csi_driver/cmd/nbs-csi-driver/nbs-csi-driver")
rm ./cloud/blockstore/tools/csi_driver/cmd/nbs-csi-driver/nbs-csi-driver
cp ${DRIVER_PATH} ./cloud/blockstore/tools/csi_driver/cmd/nbs-csi-driver/nbs-csi-driver
```

# Build docker container with csi-driver

```bash
eval $(minikube docker-env)
docker build -t nbs-csi-driver:latest cloud/blockstore/tools/csi_driver
```

# Setup k8s cluster

```bash
cd cloud/blockstore/tools/csi_driver
kubectl apply -f ./deploy/manifests/0-ns.yaml
kubectl apply -f ./deploy/manifests/1-priorityclass.yaml
kubectl apply -f ./deploy/manifests/2-rbac.yaml
kubectl apply -f ./deploy/manifests/3-csidriver.yaml
kubectl apply -f ./deploy/manifests/4-storageclass.yaml
kubectl apply -f ./deploy/manifests/5-csi-deployment.yaml
kubectl apply -f ./deploy/manifests/6-csi-daemonset.yaml
```

# copy nbsd-lightweight and blockstore-client to minikube and run them
```bash
./ya make -r cloud/blockstore/apps/server_lightweight -DFORCE_STATIC_LINKING=yes
minikube cp cloud/blockstore/apps/server_lightweight/nbsd-lightweight /home/nbsd-lightweight
minikube cp cloud/blockstore/tools/csi_driver/deploy/server.txt /home/server.txt

./ya make -r cloud/blockstore/apps/client -DFORCE_STATIC_LINKING=yes
minikube cp cloud/blockstore/apps/client/blockstore-client /home/blockstore-client
```

```bash
minikube ssh
cd /home
sudo mkdir -p /etc/nbs-server/endpoints
sudo chmod +x nbsd-lightweight
sudo chmod +x blockstore-client
sudo ./nbsd-lightweight --service local --server-file server.txt --verbose
```

# Create test pods (1st way)
# - CSI creates pv on controller;
# - CSI mounts pv on node.

```bash
kubectl apply -f ./deploy/example/pvc-fs.yaml
kubectl apply -f ./deploy/example/pod-fs.yaml

kubectl apply -f ./deploy/example/pvc-blk.yaml
kubectl apply -f ./deploy/example/pod-blk.yaml
```

# Create test pod (2nd way)
# - CSI mounts pv on node.
#
# Compute-API has to create nbs volume before and passes disk-id to volumeHandle of pv.yaml.
# Also Compute-API can pass any arguments to volumeAttributes of pv.yaml.

```bash
minikube ssh
cd /home
./blockstore-client createvolume --storage-media-kind ssd --blocks-count 262144 --block-size 4096 --disk-id my-nbs-volume-id
```

```bash
kubectl apply -f ./deploy/example_pv/pvc.yaml
kubectl apply -f ./deploy/example_pv/pv.yaml
kubectl apply -f ./deploy/example_pv/pod.yaml
```

===========================================

# Useful commands

```bash
kubectl get nodes
kubectl -n nbs get all
kubectl -n nbs get pods
kubectl -n nbs describe node
kubectl -n nbs describe pod/nbs-csi-driver-node-xxxxx
kubectl -n nbs logs pod/nbs-csi-driver-node-xxxxx
kubectl -n nbs logs pod/nbs-csi-driver-node-xxxxx -c csi-nbs-driver
kubectl -n nbs logs pod/nbs-csi-driver-controller-xxxxx -c csi-nbs-driver
kubectl -n nbs delete deployment.apps/nbs-csi-driver-controller
kubectl get CSIDriver
kubectl get --raw "/api/v1/nodes/minikube/proxy/stats/summary"
minikube ssh
  docker ps
  docker exec -it $MYCONTAINER /bin/bash
```
