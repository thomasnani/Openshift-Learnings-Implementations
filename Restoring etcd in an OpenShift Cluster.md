<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# Restoring etcd in an OpenShift Cluster

Restoring etcd is a critical operation to recover cluster state. The following procedure covers two methods: using Velero/OADP with NetBackup CSI snapshots, and a manual etcdctl-based restore.

***

## **1. Pre-Restore Preparation**

1. **Isolate the Cluster**
    - Quarantine network access to prevent new writes.
    - Scale down or stop application workloads to reduce inconsistencies.
2. **Identify the Backup**
    - Determine the **backup name** (e.g., `etcd-backup`) and **VolumeSnapshot** IDs stored in NetBackup.
    - Note the timestamp and ensure retention policy covers it.
3. **Provision a Recovery Control Plane**
    - Prepare a temporary or standby master node (or small cluster) matching the original version.
    - Ensure `oc` client is configured to point to this recovery control plane.

***

## **2. Restoring via Velero/OADP**

This approach leverages the CSI Flex snapshots orchestrated by Velero.

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

    - `--restore-volumes` triggers CSI snapshot reprovisioning.
    - `--include-resources` ensures cluster CRDs and resources are rehydrated.
3. **Monitor Restore Progress**

```bash
oc get restores.velero.io -n oadp
oc describe restore etcd-restore -n oadp
```

    - Confirm that all VolumeSnapshots transition to `ReadyToUse`.
    - Ensure no errors in Velero pod logs:

```bash
oc logs deploy/oadp-velero -n oadp
```

4. **Restart the Control Plane**
After volumes are restored, restart etcd and API server pods:

```bash
oc scale --replicas=0 deployment/etcd -n openshift-etcd
oc scale --replicas=1 deployment/etcd -n openshift-etcd
oc rollout restart deployment/apiserver -n openshift-apiserver
```

    - Verify the etcd cluster health:

```bash
oc exec -n openshift-etcd etcd-<pod> -- etcdctl endpoint health
```


***

## **3. Manual etcdctl-Based Restore**

Use this method if Velero is unavailable or for point-in-time recovery.

1. **Retrieve the etcd Snapshot**
    - Export the snapshot from NetBackup to a local path (e.g., `/backups/etcd-snapshot.db`).
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
    - Update `/etc/etcd/etcd.conf` or the systemd unit to point to the restored data directory.
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


***

## **4. Post-Restore Verification**

- **Cluster API Access:** Ensure `oc status` succeeds without errors.
- **Data Consistency:** Confirm key ConfigMaps, Secrets, and custom resources exist.
- **Application Workloads:** Scale up deployments and verify they reach ready state.
- **Monitoring \& Alerts:** Check Prometheus/OpenShift monitoring dashboards for anomalies.

***

Maintaining tested, documented restore procedures ensures a swift and reliable recovery of your OpenShift control plane. Perform regular restore drills using both automated (Velero) and manual (etcdctl) methods to validate your NetBackup policies and operational readiness.

