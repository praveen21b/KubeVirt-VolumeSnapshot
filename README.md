# KubeVirt-VolumeSnapshot

## install KubeVirt
```bash
# Deploy KubeVirt
export KUBEVIRT_VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
echo $KUBEVIRT_VERSION
kubectl create -f "https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-operator.yaml"

kubectl create -f "https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-cr.yaml"
```

## install CDI
```bash
export TAG=$(curl -s -w %{redirect_url} https://github.com/kubevirt/containerized-data-importer/releases/latest)
export VERSION=$(echo ${TAG##*/})
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-operator.yaml
kubectl create -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-cr.yaml
```

## install CSI driver
#### I am using rook-ceph for CSI and storageClass

## install VolumeSnapshot CRDs
Before attempting to install VolumeSnapshot CRDs, it is important to confirm that the CRDs are not already present on the system. To do this, run the following command:
```bash
kubectl api-resources | grep volumesnapshot
```
If CRDs are already present, the output should be similar to the output displayed below. The second column displays the version of the CRD installed (v1beta1 in this case). Ensure that it is the correct version required by the CSI driver being used.
```bash
volumesnapshotclasses    snapshot.storage.k8s.io/v1beta1        false        VolumeSnapshotClass
volumesnapshotcontents   snapshot.storage.k8s.io/v1beta1        false        VolumeSnapshotContent
volumesnapshots          snapshot.storage.k8s.io/v1beta1        true         VolumeSnapshot
```
Installing CRDs
```bash
##export VS_VERSION=
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/refs/heads/release-8.2/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/refs/heads/release-8.2/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/refs/heads/release-8.2/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
```
## VolumeSnapshotClass
```bash
kubectl get csidrivers
NAME                          ATTACHREQUIRED   PODINFOONMOUNT   STORAGECAPACITY   TOKENREQUESTS   REQUIRESREPUBLISH   MODES        AGE
kubeops.cephfs.csi.ceph.com   true             false            false             <unset>         false               Persistent   18h
kubeops.rbd.csi.ceph.com      true             false            false             <unset>         false               Persistent   18h
```
```yaml
cat << EOF >> volumesnapshotclass.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-rbd-snapclass
driver: rook-ceph.rbd.csi.ceph.com
deletionPolicy: Delete
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-cephfs-snapclass
driver: rook-ceph.cephfs.csi.ceph.com
deletionPolicy: Delete
EOF
```

```bash
kubectl apply -f volumesnapshotclass.yaml
kubectl get volumesnapshotclass
```
## enable feature gate
```yaml
kubectl edit kubevirt kubevirt -n kubevirt
```
```bash
spec:
  configuration:
    developerConfiguration:
      featureGates:
      - Snapshot
```
## deploy a vm
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-ubuntu-datavolume
  name: vm-ubuntu-datavolume
  namespace: test
spec:
  runStrategy: Always # or Halted
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-ubuntu-datavolume
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: datavolumedisk1
          - name: cloudinitdisk
            disk:
              bus: virtio
        resources:
          requests:
            memory: 500M
      volumes:
      - dataVolume:
          name: ubuntu-dv
        name: datavolumedisk1
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |-
            #cloud-config
            chpasswd:
              list: |
                ubuntu:ubuntu
                root:toor
              expire: False
            ssh_pwauth: True
            disable_root: false
  dataVolumeTemplates:
  - metadata:
      name: ubuntu-dv
    spec:
      pvc:
        accessModes: ["ReadWriteMany"]
        volumeMode: "Filesystem"
        resources:
          requests:
            storage: 3Gi
        storageClassName: rook-cephfs
      source:
        http:
          url: "https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img"
```
```bash
 virtctl stop vm-ubuntu-datavolume -n test
```
## create snapshot object
```yaml
cat << EOF >> snap-vm.yaml
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineSnapshot
metadata:
  name: snap-vm-ubuntu-datavolume
  namespace: test
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: vm-ubuntu-datavolume
EOF
```
```bash
kubectl apply -f snap-vm.yaml
```
```bash
$ kubectl get vmsnapshot -n test
NAME                        SOURCEKIND       SOURCENAME             PHASE       READYTOUSE   CREATIONTIME   ERROR
snap-vm-ubuntu-datavolume   VirtualMachine   vm-ubuntu-datavolume   Succeeded   true         87s

```
## restore vm
```bash
cat << EOF >> restore-vm.yaml
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineRestore
metadata:
  name: restore-vm-ubuntu-datavolume
  namespace: test
spec:
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: vm-ubuntu-datavolume
  virtualMachineSnapshotName: snap-vm-ubuntu-datavolume
EOF
```
