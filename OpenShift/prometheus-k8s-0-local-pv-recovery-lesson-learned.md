# 🟢 Lesson Learned — Prometheus `prometheus-k8s-0` / Local PV Recovery

> **Incident:** Prometheus pods stuck in `Init:0/1` because the local Prometheus PV was being mounted as **ext4** while the real disk filesystem was **XFS**.  
> **Affected storage:** `ls-prometheus-data2` on `master1`; `ls-prometheus-data3` on `master2`.  
> **Final result:** `prometheus-k8s-0` and `prometheus-k8s-1` both reached **6/6 Running**.

---

## 🚨 1. Symptom

Initial monitoring check showed:

```bash
oc get pods -n openshift-monitoring
```

Prometheus was stuck:

```text
prometheus-k8s-0   0/6   Init:0/1
prometheus-k8s-1   0/6   Init:0/1
```

The affected pod was scheduled on:

```text
master1.nokia-ncp-lab-cwla.nokialab.eu
```

---

## 🔎 2. First Diagnostic — Describe the Pod

Run:

```bash
oc describe pod prometheus-k8s-0 -n openshift-monitoring
```

The important event was:

```text
MountVolume.MountDevice failed for volume "ls-prometheus-data2"

failed to mount device /dev/sdd1
fstype: ext4

mount -t ext4 -o defaults /dev/sdd1 ...
```

The kubelet then reported:

```text
wrong fs type, bad option, bad superblock on /dev/sdd1
```

### 🎯 Root cause identified

Kubernetes was attempting:

```text
/dev/sdd1 → ext4
```

but the actual partition was XFS.

This was a **filesystem-type mismatch**, not a Prometheus configuration problem.

---

## 🔗 3. Trace the Storage Chain

Follow the relationship:

```text
prometheus-k8s-0
      ↓
prometheus-k8s-db-prometheus-k8s-0
      ↓
ls-prometheus-data2
      ↓
/dev/sdd1
      ↓
XFS filesystem
```

Commands used:

```bash
oc get pvc -n openshift-monitoring
oc describe pvc prometheus-k8s-db-prometheus-k8s-0 -n openshift-monitoring
oc get pv ls-prometheus-data2 -o yaml
oc describe pv ls-prometheus-data2
```

---

## 🖥️ 4. Verify the Real Disk on master1

SSH to the node:

```bash
ssh -i id_ed25519 core@master1.nokia-ncp-lab-cwla.nokialab.eu
sudo bash
```

Verify filesystem:

```bash
blkid /dev/sdd1
```

The disk was identified as:

```text
TYPE="xfs"
PARTLABEL="var-lib-prometheus-data"
```

Verify the persistent partition label:

```bash
ls -l /dev/disk/by-partlabel/
readlink -f /dev/disk/by-partlabel/var-lib-prometheus-data
```

Result:

```text
/dev/disk/by-partlabel/var-lib-prometheus-data -> /dev/sdd1
```

This confirmed that the existing data partition was XFS and had a stable PARTLABEL path.

---

# 🧠 5. Why the Mount Failed

The original PV configuration did not explicitly specify the real filesystem:

```yaml
local:
  path: /dev/sdd1
```

Because the filesystem type was not correctly represented in the PV, kubelet attempted the mount as:

```text
fstype: ext4
```

The actual disk was:

```text
XFS
```

Therefore:

```text
ext4 mount attempt
      ↓
XFS filesystem
      ↓
wrong fs type
      ↓
FailedMount
      ↓
Prometheus remains in Init
```

---

# 🛡️ 6. Data-Safety Rule

### ❌ DO NOT FORMAT THE PROMETHEUS DISK

Never run:

```bash
mkfs.xfs /dev/sdd1
```

or another `mkfs` command on the existing data partition.

The objective was to repair the **Kubernetes PV definition**, not recreate the filesystem.

---

# 🧹 7. PV/PVC Cleanup

During recovery, the original PV/PVC objects became stuck in states such as:

```text
PV: Released
PVC: Terminating
PVC: Pending
```

The original PVC was:

```text
prometheus-k8s-db-prometheus-k8s-0
```

When the PVC remained stuck in `Terminating`, its protection finalizer was removed:

```bash
oc patch pvc prometheus-k8s-db-prometheus-k8s-0 \
  -n openshift-monitoring \
  --type=merge \
  -p '{"metadata":{"finalizers":[]}}'
```

### ⚠️ Use finalizer removal only when the object is genuinely stuck

This is a recovery action for a stuck object, not a normal first step.

---

# 🧩 8. Important Kubernetes Behavior — PV Source Is Immutable

One recovery attempt tried to change the existing PV source with `oc apply`.

That failed with:

```text
spec.persistentvolumesource: Forbidden:
spec.persistentvolumesource is immutable after creation
```

The attempted difference included:

```text
old path: /dev/sdd1
new path: /dev/disk/by-partlabel/var-lib-prometheus-data
new fsType: xfs
```

### ✅ Lesson

For a PV, the `spec.persistentVolumeSource` is immutable after creation.

Therefore, changing:

```text
local.path
fsType
```

is not an in-place edit.

The recovery must recreate the PV object with the correct source definition.

---

# 🗑️ 9. Remove the Stale PV Object Carefully

The PV had protection finalizers and was not immediately disappearing.

The recovery sequence included:

```bash
oc get pv ls-prometheus-data2 -o yaml | grep -i finalizer
```

When appropriate, the PV protection finalizer was removed:

```bash
oc patch pv ls-prometheus-data2 \
  -p '{"metadata":{"finalizers":null}}' \
  --type=merge
```

Then:

```bash
oc delete pv ls-prometheus-data2 --ignore-not-found --wait=false
```

### ⚠️ Critical distinction

This deletes the **Kubernetes PV object**.

It does **not** format or recreate:

```text
/dev/sdd1
```

The physical filesystem remained intact.

---

# 📝 10. Correct Local PV Definition

The corrected definition uses:

- the stable PARTLABEL path
- `fsType: xfs`
- `Retain` reclaim policy
- node affinity to the node containing the physical disk

### master1 / prometheus-k8s-0

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
                - master1.nokia-ncp-lab-cwla.nokialab.eu
```

---

# 🔁 11. Second Prometheus Replica — master2

The same storage issue existed for the second Prometheus replica on `master2`.

First verify the physical partition:

```bash
ssh -i id_ed25519 core@master2.nokia-ncp-lab-cwla.nokialab.eu
sudo bash
blkid /dev/sdb1
ls -l /dev/disk/by-partlabel/
```

The real disk was:

```text
/dev/sdb1
TYPE="xfs"
PARTLABEL="var-lib-prometheus-data"
```

The persistent label resolved to:

```text
/dev/disk/by-partlabel/var-lib-prometheus-data -> /dev/sdb1
```

The corrected PV was:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ls-prometheus-data3
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
                - master2.nokia-ncp-lab-cwla.nokialab.eu
```

---

# 🚀 12. Apply the Corrected PV

For the corrected manifest:

```bash
oc apply -f ls-prometheus-data3.yaml
```

Verify:

```bash
oc get pv ls-prometheus-data3 \
  -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,CAPACITY:.spec.capacity.storage,PATH:.spec.local.path,FSTYPE:.spec.local.fsType,NODE:.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]'
```

Successful state:

```text
ls-prometheus-data3   Bound   456Gi   /dev/disk/by-partlabel/var-lib-prometheus-data   xfs   master2...
```

---

# 🔗 13. Verify PVC Binding

For the second replica:

```bash
oc get pvc prometheus-k8s-db-prometheus-k8s-1 -n openshift-monitoring -w
```

Successful state:

```text
STATUS   Bound
VOLUME   ls-prometheus-data3
CAPACITY 456Gi
```

For the first replica, verify:

```bash
oc get pvc prometheus-k8s-db-prometheus-k8s-0 -n openshift-monitoring
```

The important condition is:

```text
STATUS: Bound
```

---

# ✅ 14. Final Prometheus Validation

Run:

```bash
oc get pods -n openshift-monitoring | grep prometheus-k8s
```

Final successful result from the incident:

```text
prometheus-k8s-0   6/6   Running   0
prometheus-k8s-1   6/6   Running   0
```

Also:

```bash
oc get sts prometheus-k8s -n openshift-monitoring
```

Final StatefulSet state:

```text
prometheus-k8s   2/2
```

---

# 🟣 15. What Actually Solved the Issue?

The key fix was:

```text
❌ PV assumed/used ext4
        ↓
🔎 Verify physical disk
        ↓
✅ Actual filesystem = XFS
        ↓
✅ Find persistent PARTLABEL
        ↓
✅ Recreate PV with:
       path = /dev/disk/by-partlabel/var-lib-prometheus-data
       fsType = xfs
       nodeAffinity = correct node
        ↓
✅ Bind PVC
        ↓
✅ Prometheus pods Running
```

---

# 💡 16. Lessons Learned

### 1. Always start with pod Events

```bash
oc describe pod <pod> -n openshift-monitoring
```

The kubelet event exposed the real problem: the mount was attempted as `ext4`.

### 2. Never trust the PV definition alone

Verify the actual node:

```bash
blkid <device>
lsblk -f
```

### 3. Local PVs are physically node-bound

The node affinity must match the node that owns the disk.

### 4. PV source fields are immutable

Do not try to change `local.path` or `fsType` on an existing PV with `oc apply`.

### 5. Preserve the data

Delete/recreate the Kubernetes object when required, but do not format the underlying Prometheus filesystem.

### 6. Validate the complete chain

```text
Pod
 ↓
PVC
 ↓
PV
 ↓
Node
 ↓
Physical partition
 ↓
Filesystem
 ↓
Mount
 ↓
Prometheus
```

---

# 🧪 17. Troubleshooting Cheat Sheet

```bash
# Prometheus pods
oc get pods -n openshift-monitoring | grep prometheus-k8s

# Pod events
oc describe pod prometheus-k8s-0 -n openshift-monitoring

# PVCs
oc get pvc -n openshift-monitoring

# PV details
oc get pv ls-prometheus-data2 -o yaml
oc describe pv ls-prometheus-data2

# Physical disk
blkid /dev/sdd1
lsblk -f

# Stable device path
ls -l /dev/disk/by-partlabel/
readlink -f /dev/disk/by-partlabel/var-lib-prometheus-data

# Stuck PVC finalizer — only when required
oc patch pvc prometheus-k8s-db-prometheus-k8s-0 \
  -n openshift-monitoring \
  --type=merge \
  -p '{"metadata":{"finalizers":[]}}'

# PV finalizer — only when required for a stuck deletion
oc patch pv ls-prometheus-data2 \
  -p '{"metadata":{"finalizers":null}}' \
  --type=merge

# Final validation
oc get pv | grep prometheus
oc get pvc -n openshift-monitoring | grep prometheus
oc get pods -n openshift-monitoring | grep prometheus-k8s
oc get sts prometheus-k8s -n openshift-monitoring
```

---

# 🏁 Final Outcome

| Layer | Before | After |
|---|---|---|
| Pod | ❌ `Init:0/1` | ✅ `6/6 Running` |
| Mount | ❌ ext4 against XFS | ✅ XFS |
| PV source | ❌ incomplete/wrong FS handling | ✅ PARTLABEL + `fsType: xfs` |
| PVC | ❌ Pending/Terminating during recovery | ✅ Bound |
| Local node mapping | ⚠️ required verification | ✅ explicit node affinity |
| Data partition | ✅ existing | ✅ preserved |
| StatefulSet | ❌ not ready | ✅ `2/2` |

---

## 🏷️ Tags

`OpenShift` `Kubernetes` `Prometheus` `LocalPV` `PV` `PVC` `XFS` `StorageTroubleshooting` `SRE` `DevOps` `LessonLearned`
