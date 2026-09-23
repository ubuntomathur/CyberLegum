# 🟢 Lesson Learned: Recovering `prometheus-k8s-0` Local PV Mount Issue

> **Incident Type:** OpenShift Monitoring / Prometheus Storage Recovery  
> **Component:** `prometheus-k8s-0`  
> **Storage Type:** Local PersistentVolume  
> **Filesystem:** XFS  
> **Recovery Theme:** Verify the real disk → verify filesystem → verify PV/PVC → correct local PV definition → preserve data → validate Prometheus

---

## 🚨 1. Problem Statement

The OpenShift Prometheus pod:

```text
prometheus-k8s-0
```

was not becoming healthy because its persistent storage could not be mounted correctly.

The affected local PV was:

```text
ls-prometheus-data2
```

The underlying partition was:

```text
/dev/sdd1
```

on the node where the local storage physically existed.

---

## 🔴 2. First Check — Prometheus Pod

Start with the workload:

```bash
oc get pods -n openshift-monitoring | grep prometheus
```

Then inspect the affected pod:

```bash
oc get pod prometheus-k8s-0 -n openshift-monitoring -o wide
```

Also inspect events:

```bash
oc describe pod prometheus-k8s-0 -n openshift-monitoring
```

### What to look for

Focus on:

- `FailedMount`
- `MountVolume`
- PVC/PV references
- node name
- volume-related errors

---

## 🟠 3. Check Prometheus PVC

List PVCs:

```bash
oc get pvc -n openshift-monitoring
```

Then inspect the Prometheus PVC:

```bash
oc describe pvc <PROMETHEUS-PVC> -n openshift-monitoring
```

The important relationship is:

```text
Prometheus Pod
      ↓
PersistentVolumeClaim
      ↓
PersistentVolume
      ↓
Local Disk / Partition
```

---

## 🟡 4. Identify the Bound PV

List PVs:

```bash
oc get pv
```

Find the affected Prometheus PV:

```bash
oc get pv | grep prometheus
```

Affected PV:

```text
ls-prometheus-data2
```

Inspect the complete object:

```bash
oc get pv ls-prometheus-data2 -o yaml
```

Also:

```bash
oc describe pv ls-prometheus-data2
```

---

# 🔎 5. Check the Actual Disk on the Node

Move to the node that owns the local disk.

```bash
ssh <PROMETHEUS-NODE>
```

Check block devices:

```bash
lsblk
```

Then check filesystem information:

```bash
lsblk -f
```

And verify the partition directly:

```bash
blkid /dev/sdd1
```

### Important finding

The partition was:

```text
/dev/sdd1
Filesystem: XFS
PARTLABEL: var-lib-prometheus-data
```

This was a critical part of the diagnosis.

---

# 🧠 6. Root-Cause Analysis

The original local PV definition referenced the device:

```yaml
local:
  path: /dev/sdd1
```

but the filesystem type was not explicitly defined.

The physical partition was confirmed to be:

```text
XFS
```

The partition also had a persistent label:

```text
var-lib-prometheus-data
```

So the recovery focused on making the PV definition match the **real storage characteristics** instead of changing or formatting the disk.

---

# ✅ 7. Verify the Stable Device Path

Check the partition-label links:

```bash
ls -l /dev/disk/by-partlabel/
```

Confirm the expected label points to the real partition:

```bash
readlink -f /dev/disk/by-partlabel/var-lib-prometheus-data
```

Expected target:

```text
/dev/sdd1
```

This gives a more descriptive and persistent device path for the PV.

---

# 🛡️ 8. Data-Safety Rule

## ❌ DO NOT FORMAT THE EXISTING PARTITION

Do **not** run:

```bash
mkfs.xfs /dev/sdd1
```

and do not run any other `mkfs` command against an existing Prometheus data partition.

The goal is:

> **repair the Kubernetes storage definition without destroying the Prometheus data.**

---

# 🧰 9. Preserve the Underlying Data

Before changing the PV object, confirm:

```bash
lsblk -f
blkid /dev/sdd1
```

The objective is to make sure the original filesystem is still present.

The storage itself is not recreated.

---

# ⚙️ 10. Stop Prometheus Before Storage Recovery

Scale the Prometheus StatefulSet down before rebuilding the local PV object:

```bash
oc scale statefulset prometheus-k8s -n openshift-monitoring --replicas=0
```

Verify:

```bash
oc get pods -n openshift-monitoring | grep prometheus
```

This reduces the chance of the workload continually trying to mount the changing volume during recovery.

---

# 🧹 11. Remove the Incorrect PV Object

After confirming the underlying partition and data are intact, remove the Kubernetes PV object:

```bash
oc delete pv ls-prometheus-data2
```

### ⚠️ Important

This step is about removing the **Kubernetes PV object**, not formatting the physical disk.

Do not delete or recreate:

```text
/dev/sdd1
```

---

# 📝 12. Recreate the Local PV Correctly

Create a manifest:

```bash
vi ls-prometheus-data2.yaml
```

Use the corrected local PV definition, adjusting the storage class and node name to your environment:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ls-prometheus-data2
spec:
  capacity:
    storage: 456Gi

  volumeMode: Filesystem

  accessModes:
    - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  storageClassName: lsc-prometheus-data

  local:
    path: /dev/disk/by-partlabel/var-lib-prometheus-data
    fsType: xfs

  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - master1
```

---

# 🟢 13. What Was Corrected?

### ❌ Old-style reference

```yaml
local:
  path: /dev/sdd1
```

### ✅ Corrected local storage definition

```yaml
local:
  path: /dev/disk/by-partlabel/var-lib-prometheus-data
  fsType: xfs
```

### ✅ Node affinity

The PV is explicitly associated with the node where the local disk exists.

This is important because local storage is physically tied to a specific Kubernetes node.

---

# 🚀 14. Apply the Rebuilt PV

Apply the manifest:

```bash
oc apply -f ls-prometheus-data2.yaml
```

Check the PV:

```bash
oc get pv ls-prometheus-data2
```

Then:

```bash
oc describe pv ls-prometheus-data2
```

---

# 🔗 15. Verify the PVC Rebind

Check all monitoring PVCs:

```bash
oc get pvc -n openshift-monitoring
```

Inspect the Prometheus PVC:

```bash
oc describe pvc <PROMETHEUS-PVC> -n openshift-monitoring
```

Target state:

```text
STATUS: Bound
```

---

# ▶️ 16. Start Prometheus Again

Scale the StatefulSet back:

```bash
oc scale statefulset prometheus-k8s -n openshift-monitoring --replicas=2
```

Watch the pod:

```bash
oc get pods -n openshift-monitoring -w
```

---

# ✅ 17. Final Prometheus Validation

Check:

```bash
oc get pods -n openshift-monitoring | grep prometheus
```

Then specifically:

```bash
oc get pod prometheus-k8s-0 -n openshift-monitoring -o wide
```

The successful final state was:

```text
prometheus-k8s-0    6/6    Running
```

Also verify pod events:

```bash
oc describe pod prometheus-k8s-0 -n openshift-monitoring
```

There should be no continuing volume mount failures.

---

# 🔬 18. Final Disk Verification

On the node:

```bash
lsblk -f
```

Then:

```bash
blkid /dev/sdd1
```

Confirm:

```text
TYPE="xfs"
PARTLABEL="var-lib-prometheus-data"
```

Also verify:

```bash
readlink -f /dev/disk/by-partlabel/var-lib-prometheus-data
```

Expected:

```text
/dev/sdd1
```

---

# 🧭 19. Complete Troubleshooting Flow

```text
🔴 prometheus-k8s-0 unhealthy
             │
             ▼
🔎 Check Pod Events
             │
             ▼
🔎 Check Prometheus PVC
             │
             ▼
🔎 Identify PV: ls-prometheus-data2
             │
             ▼
🖥️ Inspect node disk
             │
             ▼
🔎 lsblk -f / blkid /dev/sdd1
             │
             ▼
📌 Confirm filesystem = XFS
             │
             ▼
📌 Confirm PARTLABEL
             │
             ▼
🛑 Scale Prometheus down
             │
             ▼
🧹 Remove incorrect PV object
             │
             ▼
📝 Recreate PV
   ├── stable PARTLABEL path
   ├── fsType: xfs
   ├── Retain policy
   └── nodeAffinity
             │
             ▼
✅ PVC Bound
             │
             ▼
▶️ Scale Prometheus up
             │
             ▼
🎯 prometheus-k8s-0 = 6/6 Running
```

---

# 💡 20. Lesson Learned

### Lesson 1 — Start from the workload

Do not immediately modify the PV.

First establish:

```text
Pod → PVC → PV → Node → Disk
```

### Lesson 2 — Always verify the physical filesystem

Use:

```bash
lsblk -f
blkid /dev/sdd1
```

The Kubernetes object may not tell the complete storage story.

### Lesson 3 — Local storage is node-specific

A local PV must be scheduled with awareness of the node holding the physical disk.

### Lesson 4 — Do not destroy data while fixing metadata

The recovery objective was to repair the Kubernetes storage definition while preserving the existing XFS filesystem.

### Lesson 5 — Validate all layers after recovery

Check:

```text
PV → PVC → Pod → Mount → Filesystem → Application
```

---

# 📋 21. Useful Command Cheat Sheet

```bash
# Pod
oc get pods -n openshift-monitoring | grep prometheus

# Pod details
oc describe pod prometheus-k8s-0 -n openshift-monitoring

# PVC
oc get pvc -n openshift-monitoring

# PV
oc get pv
oc describe pv ls-prometheus-data2

# Node/Disk
lsblk -f
blkid /dev/sdd1

# Partition-label path
ls -l /dev/disk/by-partlabel/
readlink -f /dev/disk/by-partlabel/var-lib-prometheus-data

# Stop Prometheus
oc scale statefulset prometheus-k8s -n openshift-monitoring --replicas=0

# Recreate PV
oc apply -f ls-prometheus-data2.yaml

# Start Prometheus
oc scale statefulset prometheus-k8s -n openshift-monitoring --replicas=2

# Final validation
oc get pods -n openshift-monitoring | grep prometheus
```

---

# 🏁 Resolution Summary

| Area | Before | Recovery |
|---|---|---|
| Prometheus | ❌ `prometheus-k8s-0` unhealthy | ✅ Running |
| PV | ❌ Incorrect local PV definition | ✅ Recreated |
| Disk | ✅ Existing data disk | ✅ Preserved |
| Filesystem | XFS | ✅ Explicitly configured |
| Local path | Device path | ✅ PARTLABEL path |
| Node placement | Local disk dependency | ✅ Node affinity |
| PVC | Storage mount problem | ✅ Bound |
| Final pod | ❌ Not healthy | ✅ `6/6 Running` |

---

## 🏷️ Tags

`OpenShift` `Kubernetes` `Prometheus` `LocalPV` `PersistentVolume` `PersistentVolumeClaim` `XFS` `StorageTroubleshooting` `LinuxStorage` `DevOps` `SRE` `LessonLearned`
