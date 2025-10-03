# NetBackup on OpenShift: Comprehensive Implementation and Restore Guide

**Key Takeaway:**
By integrating Veritas NetBackup with an OpenShift cluster via the CSI Flex plugin and the OpenShift API for Data Protection (OADP), you can centrally manage backups of etcd, persistent volumes, and application manifests, and recover your control plane using both automated and manual methods.

---

## Table of Contents
1. Prerequisites and Architecture Overview
2. Installing the CSI Flex Plugin
3. Configuring StorageClasses and VolumeSnapshotClasses
4. Installing and Configuring OADP (Velero)
5. Defining Backup Policies in NetBackup
6. Taking Backups
   - etcd Snapshot
   - Persistent Volume Data
   - Application Manifests
7. Restoring Backups
   - Restoring etcd via Velero/OADP
   - Manual etcdctl-Based Restore
   - Restoring PV Data
   - Restoring Application Manifests
8. Validation and Monitoring
9. Best Practices

---

## 1. Prerequisites and Architecture Overview

**Cluster Requirements**
- OpenShift Container Platform 4.12 or later
- Kubernetes CSI Flex plugin support
- NetBackup 9.x with Kubernetes plugin license
- Network connectivity between OpenShift nodes and NetBackup media server
- Persistent storage (NFS, CIFS, or S3/GCS compatible) for snapshots

**Components**
- **CSI Flex Driver:** Installs on each node to intercept Kubernetes volume operations.
- **NetBackup Media Server:** Central backup controller with storage and policy engine.
- **OADP Operator:** Manages Velero under the hood to orchestrate backups.

**Architecture**
1. CSI Flex driver intercepts PV snapshot/restore calls.
2. Velero/OADP coordinates cluster-wide backup jobs (etcd, PVs, manifests).
3. NetBackup media server stores snapshots according to defined policies.

---

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

---

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

---

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

---

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

---

## 6. Taking Backups

### 6.1 etcd Snapshot
```bash
oc create backup etcd-backup \
  --include-resources=configmaps,namespace,clusterrole,clusterrolebinding,etcdclusters \
  --storage-location=nb-bsl \
  --snapshot-volumes \
  -n oadp
```

### 6.2 Persistent Volume Data
Velero automatically triggers CSI snapshots of PVs matching PVCs in backup.
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

---

## 7. Restoring Backups

### 7.1 Restoring etcd via Velero/OADP
1. **List Available Backups**
   ```bash
   oc get backups.velero.io -n oadp
   ```
2. **Initiate the Restore**
   ```bash
   oc create restore etcd-restore \
     --from-backup=etcd-backup \
     --restore-volumes \
     --include-resources=pods,services,configmaps,etcdclusters \
     -n oadp
   ```
3. **Monitor Restore Progress**
   ```bash
   oc get restores.velero.io -n oadp
   oc describe restore etcd-restore -n oadp
   oc logs deploy/oadp-velero -n oadp
   ```
4. **Restart the Control Plane**
   ```bash
   oc scale --replicas=0 deployment/etcd -n openshift-etcd
   oc scale --replicas=1 deployment/etcd -n openshift-etcd
   oc rollout restart deployment/apiserver -n openshift-apiserver
   oc exec -n openshift-etcd etcd-<pod> -- etcdctl endpoint health
   ```

### 7.2 Manual etcdctl-Based Restore
1. **Retrieve the etcd Snapshot**
   Export the snapshot from NetBackup to a local path (`/backups/etcd-snapshot.db`).
2. **Stop etcd and API Server**
   ```bash
   systemctl stop atomic-openshift-master-api
   systemctl stop atomic-openshift-master-controllers
   systemctl stop etcd
   ```
3. **Restore the Snapshot**
   ```bash
   ETCD_DATA_DIR=/var/lib/etcd
   rm -rf ${ETCD_DATA_DIR}/*
   etcdctl snapshot restore /backups/etcd-snapshot.db \
     --data-dir ${ETCD_DATA_DIR} \
     --name etcd-restore \
     --initial-advertise-peer-urls https://<host-ip>:2380 \
     --initial-cluster etcd-restore=https://<host-ip>:2380 \
     --initial-cluster-token etcd-cluster-1 \
     --initial-cluster-state new
   ```
4. **Reconfigure Service Files**
   Update `/etc/etcd/etcd.conf` or the systemd unit to point to the restored data directory.
5. **Restart Services**
   ```bash
   systemctl daemon-reload
   systemctl start etcd
   systemctl start atomic-openshift-master-controllers
   systemctl start atomic-openshift-master-api
   ```
6. **Validate Cluster Health**
   ```bash
   etcdctl endpoint health
   oc get nodes
   oc get pods --all-namespaces
   ```

### 7.3 Restoring PV Data and Application Manifests
```bash
oc create restore pv-restore \
  --from-backup=etcd-backup \
  --restore-volumes \
  -n oadp
oc create restore app-restore \
  --from-backup=app-manifest-backup \
  --include-namespaces=my-app-namespace \
  -n oadp
```

---

## 8. Validation and Monitoring
- **NetBackup Activity Monitor:** Track policy success/failure.
- **Velero Logs:**
  ```bash
  oc logs deploy/oadp-velero -n oadp
  ```
- **VolumeSnapshotStatus:**
  ```bash
  oc describe volumesnapshot <snapshot-name> -n <ns>
  ```

---

## 9. Best Practices
- **Isolate Backup Storage:** Use dedicated network/storage for snapshots.
- **Encrypt Data in Transit:** Enable TLS between CSI driver and media server.
- **Test Restores Regularly:** Quarterly drills for full cluster restore.
- **Rotate Credentials:** Manage NetBackup secrets via OpenShift Secrets.
- **Monitor Retention:** Align policy retention with compliance requirements.

---

_With this document, share and customize your NetBackup strategy for OpenShift clusters, ensuring reliable backups and restores for your team._