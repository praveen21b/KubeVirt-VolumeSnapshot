# KubeVirt-VolumeSnapshot

## install KubeVirt
```bash
# Deploy KubeVirt
export KUBEVIRT_VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
echo $KUBEVIRT_VERSION
kubectl create -f "https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-operator.yaml"

kubectl create -f "https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-cr.yaml"
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
