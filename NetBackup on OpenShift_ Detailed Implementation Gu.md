<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# NetBackup on OpenShift: Detailed Implementation Guide

**Key Takeaway:**
By integrating Veritas NetBackup with an OpenShift cluster via the CSI Flex plugin and the OpenShift API for Data Protection (OADP), you can centrally manage backups of etcd, persistent volumes, and application manifests. This guide walks through prerequisites, installation, configuration, policy definition, and real-time backup and restore procedures.

***

## 1. Prerequisites and Architecture Overview

**Cluster Requirements**

- OpenShift Container Platform 4.12 or later
- Kubernetes CSI Flex plugin support
- NetBackup 9.x with Kubernetes plugin license
- Network connectivity between OpenShift nodes and NetBackup media server
- Persistent storage (NFS, CIFS, or S3/GCS compatible) for snapshots

**Components**

- **CSI Flex Driver**: Installs on each node to intercept Kubernetes volume operations.
- **NetBackup Media Server**: Central backup controller with storage and policy engine.
- **OADP Operator**: Manages Velero under the hood to orchestrate backups.

Architecture:

1. CSI Flex driver intercepts PV snapshot/restore calls.
2. Velero/OADP coordinates cluster-wide backup jobs (etcd, PVs, manifests).
3. NetBackup media server stores snapshots according to defined policies.

***

## 2. Installing the CSI Flex Plugin

1. **Obtain the YAMLs**
Download the NetBackup CSI Flex driver manifests from Veritas support portal.
2. **Deploy the Driver**

```bash
oc apply -f nb-csi-flex-driver-namespace.yaml
oc apply -f nb-csi-flex-driver-sa.yaml
oc apply -f nb-csi-flex-driver-rbac.yaml
oc apply -f nb-csi-flex-driver-crd.yaml
oc apply -f nb-csi-flex-driver-daemonset.yaml
```

3. **Verify Installation**

```bash
oc get pods -n netbackup-csi
oc get csidrivers
```


***

## 3. Configuring StorageClasses and VolumeSnapshotClasses

1. **Define a VolumeSnapshotClass**

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: netbackup-snapclass
driver: com.netbackup.csi.flex
deletionPolicy: Delete
parameters:
  nbServer: nb-media.example.com
  policyName: etcd-backup-policy
```

2. **Define a StorageClass (optional for dynamic PV)**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: netbackup-storage
provisioner: com.netbackup.csi.flex
parameters:
  nbServer: nb-media.example.com
  policyName: data-pv-backup-policy
reclaimPolicy: Delete
volumeBindingMode: Immediate
```


***

## 4. Installing and Configuring OADP (Velero)

1. **Install OADP Operator**

```bash
oc apply -f https://raw.githubusercontent.com/openshift/oadp-operator/main/deploy/oadp-operator.yaml
```

2. **Create NetBackup Secret**

```bash
oc create secret generic netbackup-credentials \
  --from-literal=NB_USER=admin \
  --from-literal=NB_PASS='YourPassword' \
  -n openshift-operators
```

3. **Deploy Velero Custom Resource**

```yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: oadp
  namespace: openshift-operators
spec:
  backupStorageLocations:
  - name: nb-bsl
    provider: csi
    csi:
      # Use the CSI Flex driver
      driver: com.netbackup.csi.flex
      volumeSnapshotLocation: netbackup-snapclass
    objectStorage:
      bucket: cluster-backups
      prefix: oadp
  configuration:
    noobaa: {}
  backupRepositories:
  - name: nb-repo
    provider: csi
    csi:
      driver: com.netbackup.csi.flex
      volumeSnapshotLocation: netbackup-snapclass
```

4. **Verify Velero Deployment**

```bash
oc get pods -n oadp
oc get backups.velero.io -n oadp
```


***

## 5. Defining Backup Policies in NetBackup

1. **Access NetBackup Admin Console**
    - Navigate to **Policies → New**.
2. **Create Policy “etcd-backup-policy”**
    - Type: **Snapshot**
    - Add the OpenShift master node host as client.
    - Schedule: **Daily at 02:00**; Retention: **7 days**.
3. **Create Policy “data-pv-backup-policy”**
    - Type: **Snapshot**
    - Client: All OpenShift worker nodes.
    - Schedule: **Hourly**; Retention: **24 hours**.
4. **Create Policy “manifest-backup-policy”**
    - Type: **Framework Backup** via Velero hooks.
    - Include Kubernetes resources: Namespaces, CRDs, Deployments, Services, ConfigMaps, Secrets.

***

## 6. Taking Backups

### 6.1 etcd Snapshot

```bash
# Initiate etcd backup via Velero
oc create backup etcd-backup \
  --include-resources=configmaps,namespace,clusterrole,clusterrolebinding,etcdclusters \
  --storage-location=nb-bsl \
  --snapshot-volumes \
  -n oadp
```

- Verifies snapshot created under policy `etcd-backup-policy`.


### 6.2 Persistent Volume Data

- Velero automatically triggers CSI snapshots of PVs matching PVCs in backup.
- Confirm snapshots:

```bash
oc get volumesnapshots -A
```


### 6.3 Application Manifests

```bash
oc create backup app-manifest-backup \
  --include-namespaces=my-app-namespace \
  --storage-location=nb-bsl \
  -n oadp
```

- Exports YAMLs to NetBackup repository under `manifest-backup-policy`.

***

## 7. Restoring Backups

### 7.1 Restoring etcd

1. **Prepare New Cluster (or single-master recovery).**
2. **Apply Snapshot to etcd**

```bash
oc create restore etcd-restore \
  --from-backup=etcd-backup \
  --restore-volumes \
  -n oadp
```

3. **Restart control plane pods** to pick up restored etcd data.

### 7.2 Restoring PV Data

```bash
oc create restore pv-restore \
  --from-backup=etcd-backup \
  --restore-volumes \
  -n oadp
```

- CSI driver reprovisions PVs from snapshots; rebinds PVCs.


### 7.3 Restoring Application Manifests

```bash
oc create restore app-restore \
  --from-backup=app-manifest-backup \
  --include-namespaces=my-app-namespace \
  -n oadp
```

- Recreates Deployments, Services, ConfigMaps, and Secrets.

***

## 8. Validation and Monitoring

- **NetBackup Activity Monitor**: Track policy success/failure.
- **Velero Logs**:

```bash
oc logs deploy/oadp-velero -n oadp
```

- **VolumeSnapshotStatus**:

```bash
oc describe volumesnapshot <snapshot-name> -n <ns>
```


***

## 9. Best Practices

- **Isolate Backup Storage**: Use dedicated network/storage for snapshots.
- **Encrypt Data in Transit**: Enable TLS between CSI driver and media server.
- **Test Restores Regularly**: Quarterly drills for full cluster restore.
- **Rotate Credentials**: Manage NetBackup secrets via OpenShift Secrets.
- **Monitor Retention**: Align policy retention with compliance requirements.

***

_With this comprehensive guide, you can deploy and manage NetBackup-driven backups for your OpenShift clusters at scale. Customize schedules, retention, and resources to match your SLAs before sharing with your team._

