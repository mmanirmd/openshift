# OpenShift Backup & Restore Runbook (On-Prem)
### OADP (Velero) + S3 Object Storage

**Audience:** Platform engineers, including junior team members running this for the first time.
**Scope:** On-premises OpenShift Container Platform 4.x cluster. Backup artifacts stored in an S3 or S3-compatible bucket.
**Assumed knowledge:** Basic `oc` CLI usage and YAML.

> **Before you start:** fill in the *Environment Variables* table in Section 1.3. Every command in this guide uses those placeholder values.

---

## Table of Contents

| Part | Topic |
|---|---|
| 0 | What to back up, and with which tool |
| 1 | Prerequisites and S3 bucket preparation |
| 2 | Install the OADP Operator |
| 3 | Create the S3 credentials Secret |
| 4 | Create the DataProtectionApplication (DPA) |
| 5 | Configure CSI VolumeSnapshotClass |
| 6 | Choose the right volume backup method |
| 7 | Your first test backup |
| 8 | Scheduled production backups |
| 9 | Restore procedures (5 scenarios) |
| 10 | etcd / control-plane backup and restore |
| 11 | Monitoring and alerting |
| 12 | DR drill checklist |
| 13 | Troubleshooting |
| 14 | Command cheat sheet |

---

# Part 0 — What to back up, and with which tool

This is the most important section. Read it before touching the cluster.

## 0.1 The three layers of an OpenShift backup

No single tool backs up an entire OpenShift cluster. You need three separate mechanisms, and they protect different things.

| Layer | What it contains | Tool | Covered in |
|---|---|---|---|
| **Layer 1: Cluster state** | etcd — every API object in the cluster, including secrets, RBAC, operator config, cluster operators | `cluster-backup.sh` (etcd snapshot) | Part 10 |
| **Layer 2: Application workloads** | Namespaced resources + persistent volume data for your apps | **OADP / Velero** | Parts 1–9 |
| **Layer 3: Cluster configuration as code** | Install config, MachineConfigs, operator subscriptions, network policy, IDP config | **Git + GitOps (Argo CD)** | Section 0.4 |

**Critical limitation to understand up front:** Red Hat does not support full cluster backup and restore with OADP. OADP is for customer workload namespaces and the cluster-scoped resources those namespaces depend on. It is **not** a cluster rebuild tool. If you lose the whole cluster, you rebuild it from your install config and GitOps repo, then use OADP to restore application namespaces and data into the new cluster.

## 0.2 What OADP backs up (Layer 2)

When you back up a namespace, Velero captures:

- All namespaced Kubernetes/OpenShift API objects in that namespace: Deployments, DeploymentConfigs, StatefulSets, DaemonSets, Jobs, CronJobs, Pods, Services, Routes, Ingresses, ConfigMaps, Secrets, ServiceAccounts, RoleBindings, PVCs, NetworkPolicies, HPAs, PodDisruptionBudgets, and any Custom Resources in the namespace.
- **Persistent volume data** — either via CSI snapshot or file-system copy into S3 (see Part 6).
- **Cluster-scoped resources referenced by those namespaces** — for example the PersistentVolume objects behind the PVCs, and (depending on settings) ClusterRoles/ClusterRoleBindings and SCCs bound to the ServiceAccounts in the namespace.
- Internal registry images (ImageStreams), if `backupImages` is enabled and the internal registry is configured.

## 0.3 What OADP does NOT back up

Do not assume these are protected. They are not:

| Not backed up | Protect it this way |
|---|---|
| Nodes, MachineConfigs, MachineSets | Git / install config, plus etcd snapshot |
| Cluster Operators and their config (Ingress, Image Registry, Authentication) | Git / GitOps, plus etcd snapshot |
| Identity provider config and the htpasswd/LDAP bind secret | Git (secret in Vault or Sealed Secrets), plus etcd snapshot |
| Cluster-wide certificates, custom CA bundles, API/Ingress certs | Git / secret manager |
| Operator Subscriptions in `openshift-operators` | GitOps repo |
| `kubeadmin` password / kubeconfig from install | Secure offline vault |
| Container images in an external registry | Registry's own replication/backup |
| etcd itself | `cluster-backup.sh` — Part 10 |
| Anything in `openshift-*` namespaces (do not try to back these up with OADP) | etcd snapshot + reinstall from config |

## 0.4 Recommended resource inventory

Use this table to decide what your team actually backs up. Fill in the owner column for your environment.

| # | Resource category | Example namespaces/objects | Method | Frequency | Retention |
|---|---|---|---|---|---|
| 1 | Tier-1 stateful apps (databases, message brokers) | `payments-prod`, `kafka-prod` | OADP + CSI snapshot + Data Mover | Every 6–12 h | 30 days |
| 2 | Tier-2 stateless apps | `api-prod`, `web-prod` | OADP, metadata only | Daily | 30 days |
| 3 | Tier-3 dev/test namespaces | `*-dev` | OADP, metadata only | Weekly | 14 days |
| 4 | Platform add-on namespaces you own | `monitoring-custom`, `logging-custom` | OADP | Daily | 30 days |
| 5 | Cluster-scoped config objects | SCCs, ClusterRoles, ClusterRoleBindings, CRDs, StorageClasses, custom PriorityClasses | OADP (dedicated schedule) | Daily | 30 days |
| 6 | etcd (full cluster state) | — | `cluster-backup.sh`, copied to S3 | Daily + before every change window | 14 days |
| 7 | Cluster install & config as code | `install-config.yaml`, MachineConfigs, Argo CD apps, operator subscriptions | Git repository | On every change | Forever (Git history) |
| 8 | Secrets that cannot be regenerated | Pull secrets, IDP bind credentials, TLS keys | Enterprise secret vault | On every change | Per policy |

## 0.5 Set your RPO and RTO first

Agree these numbers with the application owners **before** you build schedules, and write them down:

- **RPO (Recovery Point Objective):** how much data loss is acceptable. This sets your backup frequency.
- **RTO (Recovery Time Objective):** how long a restore may take. This determines whether you need CSI snapshots (fast) or file-system backup (slower but portable).

---

# Part 1 — Prerequisites and S3 bucket preparation

## 1.1 Cluster prerequisites

Verify each of these before you begin:

```bash
# 1. You have cluster-admin
oc auth can-i '*' '*' --all-namespaces
# expected: yes

# 2. Cluster version — note it, you need it to pick the OADP version
oc get clusterversion

# 3. All cluster operators healthy
oc get co | grep -v "True.*False.*False"
# expected: only the header line

# 4. Your storage class and CSI driver
oc get storageclass
oc get csidrivers

# 5. Does your CSI driver support snapshots?
oc get volumesnapshotclass
# If this returns "No resources found", you will use File System Backup (Part 6)
```

## 1.2 Pick the right OADP version

OADP ships as an OLM operator and each OADP minor version supports a specific range of OpenShift versions. Before installing:

1. Open the OADP release notes in the Red Hat docs for your OpenShift version.
2. Confirm the supported OADP version, then subscribe to the matching channel (for example `stable-1.4` or `stable-1.5`). Using the `stable` channel without pinning can pull a version that is not supported on your cluster.

> The OADP Operator installs Velero inside the cluster. You never install Velero separately — but you will install the **velero CLI** on your workstation for troubleshooting (Section 7.5).

## 1.3 Environment variables — fill this in

Copy this into your team wiki and replace every value.

| Variable | Your value | Example |
|---|---|---|
| `OADP_NAMESPACE` | | `openshift-adp` |
| `S3_BUCKET` | | `ocp-backups-prod` |
| `S3_PREFIX` | | `cluster-dfw-prod-01` |
| `S3_URL` | | `https://s3.internal.example.com` (leave blank for real AWS) |
| `S3_REGION` | | `us-east-1` |
| `S3_ACCESS_KEY` | | *(store in vault)* |
| `S3_SECRET_KEY` | | *(store in vault)* |
| `S3 uses self-signed CA?` | | yes / no |
| `DEFAULT_STORAGECLASS` | | `thin-csi` |
| `VOLUMESNAPSHOTCLASS` | | `csi-vsphere-snapshotclass` |

Export them in your shell for the rest of this guide:

```bash
export OADP_NAMESPACE=openshift-adp
export S3_BUCKET=ocp-backups-prod
export S3_PREFIX=cluster-dfw-prod-01
export S3_URL=https://s3.internal.example.com
export S3_REGION=us-east-1
```

## 1.4 Create and configure the S3 bucket

Create one bucket per cluster, or one shared bucket with a unique `prefix` per cluster. **Never point two clusters at the same bucket + prefix combination.**

Recommended bucket settings:

| Setting | Value | Why |
|---|---|---|
| Versioning | Enabled | Protects against accidental object deletion |
| Server-side encryption | Enabled (SSE-S3 or SSE-KMS) | Backups contain Secrets in plaintext YAML |
| Object Lock / WORM | Enabled if available | Ransomware protection |
| Lifecycle rule | Transition to cold tier after 30 days, expire after your retention period | Cost control |
| Public access | Blocked | Non-negotiable |
| Bucket policy | Restrict to the backup IAM user/service account only | Least privilege |

**Important:** Velero manages its own object expiry through the `ttl` field on a Backup. If you also set an aggressive S3 lifecycle expiry, you can delete objects Velero still believes exist, which produces confusing "backup exists but data is missing" failures. Set the S3 lifecycle expiry to **longer** than your longest Velero `ttl`.

## 1.5 IAM permissions

If you are on real AWS S3, the backup user needs this policy. Replace the bucket name.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:PutObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": ["arn:aws:s3:::ocp-backups-prod/*"]
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": ["arn:aws:s3:::ocp-backups-prod"]
    }
  ]
}
```

For on-prem S3 (MinIO, ODF/NooBaa, Dell ECS, NetApp StorageGRID), create a user with read/write access to the bucket and generate an access key pair.

## 1.6 Network prerequisites (on-prem gotchas)

These cause most first-time failures on on-prem clusters:

- **Firewall:** every node (not just the control plane) must reach the S3 endpoint on 443. The node agent pods that upload volume data run on worker nodes.
- **Egress proxy:** if your cluster uses a cluster-wide proxy, confirm the S3 endpoint is either reachable through the proxy or listed in `noProxy`. Check with `oc get proxy/cluster -o yaml`.
- **DNS:** nodes must resolve the S3 hostname. Test from a node:
  ```bash
  oc debug node/<any-worker> -- chroot /host curl -sSI https://s3.internal.example.com
  ```
- **Path style:** most on-prem S3 appliances require path-style addressing (`https://endpoint/bucket`) rather than virtual-host style (`https://bucket.endpoint`). You will set `s3ForcePathStyle: "true"` in Part 4.
- **Self-signed certificate:** you will need the CA bundle in base64 (Section 4.2).

---

# Part 2 — Install the OADP Operator

You can do this through the web console or the CLI. The CLI method is repeatable and can be committed to Git, so it is preferred.

## 2.1 Option A — Web console (easiest for first-timers)

1. Log in to the OpenShift web console as a cluster-admin.
2. Go to **Operators → OperatorHub**.
3. In **Filter by keyword**, type `OADP`.
4. Select **OADP Operator** (published by Red Hat) and click **Install**.
5. On the install page:
   - **Update channel:** select the channel matching your OpenShift version (for example `stable-1.4`).
   - **Installation mode:** A specific namespace on the cluster.
   - **Installed Namespace:** `openshift-adp` (created automatically).
   - **Update approval:** choose **Manual** for production. This prevents an unplanned operator upgrade from breaking your backups.
6. Click **Install** and wait for the status to become **Succeeded**.
7. Verify under **Operators → Installed Operators**.

## 2.2 Option B — CLI (recommended, GitOps-friendly)

Create `oadp-operator-install.yaml`:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-adp
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: redhat-oadp-operator-group
  namespace: openshift-adp
spec:
  targetNamespaces:
    - openshift-adp
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: redhat-oadp-operator
  namespace: openshift-adp
spec:
  channel: stable-1.4          # <-- set to the channel for YOUR OCP version
  name: redhat-oadp-operator
  source: redhat-operators     # in a disconnected cluster, your mirrored catalog name
  sourceNamespace: openshift-marketplace
  installPlanApproval: Manual  # Automatic is acceptable for non-prod
```

Apply it:

```bash
oc apply -f oadp-operator-install.yaml
```

If you set `installPlanApproval: Manual`, approve the install plan:

```bash
oc get installplan -n openshift-adp
oc patch installplan <installplan-name> -n openshift-adp \
  --type merge -p '{"spec":{"approved":true}}'
```

## 2.3 Verify the operator is running

```bash
oc get csv -n openshift-adp
# expected PHASE: Succeeded

oc get pods -n openshift-adp
# expected: openshift-adp-controller-manager-... Running
```

Do not proceed until the CSV phase is `Succeeded`.

> **Disconnected cluster note:** if your cluster has no internet access, you must mirror the OADP operator catalog into your local registry first using `oc-mirror`, then point the Subscription `source` at your mirrored CatalogSource.

---

# Part 3 — Create the S3 credentials Secret

Velero reads S3 credentials from a Secret in the OADP namespace. The file format is the standard AWS credentials format, and Velero expects the Secret key to be named `cloud`.

## 3.1 Create the credentials file

```bash
cat > /tmp/credentials-velero <<'EOF'
[default]
aws_access_key_id=YOUR_ACCESS_KEY_HERE
aws_secret_access_key=YOUR_SECRET_KEY_HERE
EOF
```

Rules for this file, each of which causes a silent failure if broken:

- The profile header must be `[default]` unless you also set `profile:` in the DPA.
- No quotes around the values.
- No spaces around the `=`.
- No trailing whitespace.

## 3.2 Create the Secret

```bash
oc create secret generic cloud-credentials \
  -n openshift-adp \
  --from-file cloud=/tmp/credentials-velero
```

## 3.3 Clean up and verify

```bash
# Remove the plaintext file immediately
shred -u /tmp/credentials-velero

# Verify the secret exists and has the 'cloud' key
oc get secret cloud-credentials -n openshift-adp -o jsonpath='{.data}' | grep -o cloud
```

> **Do not commit this Secret to Git.** If you manage the cluster with GitOps, use Sealed Secrets, External Secrets Operator, or your Vault integration.

## 3.4 Rotating the key later

```bash
oc delete secret cloud-credentials -n openshift-adp
oc create secret generic cloud-credentials -n openshift-adp \
  --from-file cloud=/tmp/credentials-velero-new
oc rollout restart deployment/velero -n openshift-adp
```

---

# Part 4 — Create the DataProtectionApplication (DPA)

The DPA is the main configuration object. Creating it causes the operator to deploy the Velero pod, the node agent DaemonSet, and register your backup storage location.

## 4.1 DPA for on-prem S3-compatible storage (most common case)

Create `dpa.yaml`:

```yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: dpa-prod
  namespace: openshift-adp
spec:
  configuration:
    velero:
      defaultPlugins:
        - openshift       # mandatory on OpenShift
        - aws             # the S3 object-store plugin (used for ALL S3-compatible storage)
        - csi             # required for CSI volume snapshots
      resourceTimeout: 10m
      podConfig:
        resourceAllocations:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: "2"
            memory: 2Gi
    nodeAgent:
      enable: true
      uploaderType: kopia   # kopia is the default and recommended uploader
      podConfig:
        resourceAllocations:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: "2"
            memory: 4Gi
  backupLocations:
    - name: default
      velero:
        provider: aws
        default: true
        credential:
          key: cloud
          name: cloud-credentials
        objectStorage:
          bucket: ocp-backups-prod
          prefix: cluster-dfw-prod-01
        config:
          region: us-east-1
          s3Url: https://s3.internal.example.com
          s3ForcePathStyle: "true"
          insecureSkipTLSVerify: "false"
```

Field-by-field explanation:

| Field | What it does |
|---|---|
| `defaultPlugins: openshift` | Handles OpenShift-specific objects (Routes, DeploymentConfigs, SCC/ImageStream handling). Always required. |
| `defaultPlugins: aws` | The object-store plugin. Use this for **any** S3-compatible endpoint, not just AWS. |
| `defaultPlugins: csi` | Enables CSI volume snapshot support. Include it if your storage supports snapshots. |
| `defaultPlugins: kubevirt` | Add **only** if you run OpenShift Virtualization and need to back up VMs. |
| `nodeAgent.enable: true` | Deploys a DaemonSet on every node, needed for file-system backup and for Data Mover. |
| `uploaderType: kopia` | The repository engine for file-system backups. Kopia is the current default; `restic` is legacy. |
| `prefix` | Folder inside the bucket. Mandatory if the bucket is shared between clusters. |
| `s3Url` | Your on-prem endpoint. **Omit this field entirely for real AWS S3.** |
| `s3ForcePathStyle: "true"` | Required by most on-prem S3 appliances. Omit for real AWS. |
| `region` | On-prem appliances often ignore it, but Velero requires a value. `us-east-1` is a safe placeholder. |

## 4.2 If your S3 endpoint uses a self-signed certificate

Get the CA certificate, base64-encode it as a single line, and add it under `objectStorage`:

```bash
# Get the cert from your S3 endpoint
openssl s_client -showcerts -connect s3.internal.example.com:443 </dev/null 2>/dev/null \
  | openssl x509 -outform PEM > /tmp/s3-ca.crt

# Base64-encode it with no line wrapping
base64 -w 0 /tmp/s3-ca.crt
```

Add the output to the DPA:

```yaml
        objectStorage:
          bucket: ocp-backups-prod
          prefix: cluster-dfw-prod-01
          caCert: <paste-the-base64-string-here>
        config:
          insecureSkipTLSVerify: "false"
```

Never set `insecureSkipTLSVerify: "true"` in production — it disables certificate validation entirely.

## 4.3 If your cluster has no internal image registry

On many on-prem clusters the internal image registry is disabled or backed by no storage. If so, add this to `spec` to prevent image backup failures:

```yaml
spec:
  backupImages: false
```

## 4.4 Optional: pin the node agent to specific nodes

By default the node agent runs on every node. To restrict it (for example, to keep it off infra nodes):

```yaml
    nodeAgent:
      enable: true
      uploaderType: kopia
      podConfig:
        nodeSelector:
          node-role.kubernetes.io/worker: ""
```

Be careful: if a workload's PVC is mounted on a node with no node agent pod, file-system backup of that volume will fail.

## 4.5 Apply and verify

```bash
oc apply -f dpa.yaml
```

Wait 1–2 minutes, then run all four checks:

```bash
# 1. DPA reconciled successfully
oc get dpa -n openshift-adp
oc get dpa dpa-prod -n openshift-adp -o jsonpath='{.status.conditions}' | jq

# 2. Pods are running: velero + one node-agent per node
oc get pods -n openshift-adp

# 3. THE MOST IMPORTANT CHECK — backup storage location must be Available
oc get backupstoragelocation -n openshift-adp
```

Expected output of the last command:

```
NAME      PHASE       LAST VALIDATED   AGE   DEFAULT
default   Available   10s              2m    true
```

If PHASE shows `Unavailable`, **stop and fix it before continuing.** Diagnose with:

```bash
oc describe backupstoragelocation default -n openshift-adp
oc logs -n openshift-adp deployment/velero | tail -50
```

See the troubleshooting table in Part 13 for the common causes.

---

# Part 5 — Configure CSI VolumeSnapshotClass

Skip this part if your storage does not support CSI snapshots. You will use File System Backup instead (Part 6).

## 5.1 Check what you have

```bash
oc get volumesnapshotclass
oc get csidrivers
```

## 5.2 Label the VolumeSnapshotClass for Velero

Velero only uses a VolumeSnapshotClass that carries a specific label. Add it:

```bash
oc label volumesnapshotclass <your-snapshot-class> \
  velero.io/csi-volumesnapshot-class="true"
```

Or define the class in full:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-vsphere-snapshotclass
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: csi.vsphere.vmware.com     # your CSI driver name
deletionPolicy: Retain
```

Use `deletionPolicy: Retain` so that deleting the VolumeSnapshot object does not immediately delete the underlying storage snapshot.

## 5.3 Verify the label took

```bash
oc get volumesnapshotclass -o custom-columns=\
NAME:.metadata.name,DRIVER:.driver,VELERO:.metadata.labels.velero\\.io/csi-volumesnapshot-class
```

## 5.4 Test that snapshots actually work

Before relying on this in a backup, prove the storage layer works:

```bash
oc new-project snapshot-test
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: snapshot-test
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
EOF

cat <<'EOF' | oc apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: test-snap
  namespace: snapshot-test
spec:
  volumeSnapshotClassName: csi-vsphere-snapshotclass
  source:
    persistentVolumeClaimName: test-pvc
EOF

oc get volumesnapshot -n snapshot-test
# READYTOUSE must become true
```

If `READYTOUSE` never becomes `true`, your CSI driver does not have working snapshot support. Use File System Backup.

Clean up: `oc delete project snapshot-test`

---

# Part 6 — Choose the right volume backup method

This decision matters more than any other in this guide. Choose per application tier.

## 6.1 The three methods

| Method | How it works | Data lands in S3? | Speed | Use when |
|---|---|---|---|---|
| **CSI snapshot only** | Storage array takes a snapshot; only metadata goes to S3 | ❌ No — snapshot stays in your storage array | Fastest | Fast local rollback on the same cluster and same storage |
| **CSI snapshot + Data Mover** | Takes a CSI snapshot, then the node agent copies the snapshot data into S3 | ✅ Yes | Medium | **Recommended default for Tier-1 apps.** Consistent and portable |
| **File System Backup (FSB)** | Node agent reads files from the mounted volume and copies them to S3 with Kopia | ✅ Yes | Slowest | Storage has no snapshot support; NFS/RWX volumes |

## 6.2 Decision guide

```
Does your CSI driver support snapshots (Part 5.4 passed)?
├─ NO  → File System Backup
└─ YES → Do you need the backup to survive loss of the storage array,
          or to restore into a different cluster?
          ├─ NO  → CSI snapshot only (fast local rollback)
          └─ YES → CSI snapshot + Data Mover   ← recommended default
```

**The trap to avoid:** a CSI-snapshot-only backup looks successful and is useless in a real disaster. If the array or the cluster is gone, so are the snapshots. For genuine disaster recovery, the data must be in S3.

## 6.3 How each is expressed in a Backup CR

```yaml
# CSI snapshot only — no extra fields needed, just have the csi plugin enabled
spec:
  includedNamespaces: [myapp-prod]

# CSI snapshot + Data Mover (data copied to S3)
spec:
  includedNamespaces: [myapp-prod]
  snapshotMoveData: true

# File System Backup (Kopia copies files to S3)
spec:
  includedNamespaces: [myapp-prod]
  defaultVolumesToFsBackup: true
```

## 6.4 Application consistency — read this for databases

Velero copies files or takes a block-level snapshot. It does **not** know your database is mid-transaction. A snapshot of a running PostgreSQL data directory is crash-consistent, not application-consistent — it usually replays fine, but not always.

For Tier-1 databases, use backup hooks to quiesce the database first:

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: postgres-prod-backup
  namespace: openshift-adp
spec:
  includedNamespaces:
    - postgres-prod
  snapshotMoveData: true
  hooks:
    resources:
      - name: pg-quiesce
        includedNamespaces:
          - postgres-prod
        labelSelector:
          matchLabels:
            app: postgresql
        pre:
          - exec:
              container: postgresql
              command:
                - /bin/bash
                - -c
                - "psql -U postgres -c 'CHECKPOINT;' && psql -U postgres -c \"SELECT pg_backup_start('velero');\""
              onError: Fail
              timeout: 60s
        post:
          - exec:
              container: postgresql
              command:
                - /bin/bash
                - -c
                - "psql -U postgres -c \"SELECT pg_backup_stop();\""
              onError: Continue
              timeout: 60s
```

Adjust the commands for your database engine. For MySQL use `FLUSH TABLES WITH READ LOCK`. For MongoDB use `fsyncLock()`.

The long-term best practice is to keep using the database's own native dump/backup tooling in addition to OADP, and let OADP protect the Kubernetes objects and the underlying volumes.

---

# Part 7 — Your first test backup

Never make your first backup a production namespace. Build a throwaway app, back it up, destroy it, restore it, and confirm the data came back.

## 7.1 Create a test application with data

```bash
oc new-project backup-test

cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-data
  namespace: backup-test
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
  namespace: backup-test
spec:
  replicas: 1
  selector:
    matchLabels: {app: test-app}
  template:
    metadata:
      labels: {app: test-app}
    spec:
      containers:
        - name: app
          image: registry.access.redhat.com/ubi9/ubi-minimal:latest
          command: ["/bin/sh","-c","sleep infinity"]
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: test-data
EOF

oc rollout status deployment/test-app -n backup-test
```

Write a file you can verify later:

```bash
POD=$(oc get pod -n backup-test -l app=test-app -o name)
oc exec -n backup-test $POD -- sh -c \
  'echo "backup-proof-$(date +%s)" > /data/proof.txt; cat /data/proof.txt'
```

**Write down the exact string it prints.** You will compare against it after the restore.

## 7.2 Run the backup

```bash
cat <<'EOF' | oc apply -f -
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: backup-test-001
  namespace: openshift-adp
spec:
  includedNamespaces:
    - backup-test
  snapshotMoveData: true        # use defaultVolumesToFsBackup: true if no CSI snapshots
  storageLocation: default
  ttl: 720h0m0s                 # 30 days
EOF
```

## 7.3 Watch it complete

```bash
# Watch the phase
oc get backup backup-test-001 -n openshift-adp -w

# Phases you will see: New → InProgress → WaitingForPluginOperations → Completed
```

Terminal phases and what they mean:

| Phase | Meaning | Action |
|---|---|---|
| `Completed` | Success | Continue |
| `PartiallyFailed` | Some items failed | Investigate — a partial backup is not a backup |
| `Failed` | Backup failed | Fix and retry |
| `FailedValidation` | The Backup spec is invalid | Check `.status.validationErrors` |

Get the detail:

```bash
oc get backup backup-test-001 -n openshift-adp -o jsonpath='{.status}' | jq
```

Check the item count and warnings in `.status.progress` and `.status.warnings`.

## 7.4 Confirm the data reached S3

Do not trust the `Completed` phase alone. Look in the bucket:

```bash
aws s3 ls s3://ocp-backups-prod/cluster-dfw-prod-01/backups/backup-test-001/ \
  --endpoint-url https://s3.internal.example.com
```

You should see files including `backup-test-001.tar.gz`, `velero-backup.json`, and resource list JSON files. If you used Data Mover or FSB, there will also be data under a `kopia/` path in the bucket.

## 7.5 Install the velero CLI (strongly recommended)

The `velero` CLI gives far better diagnostics than `oc` alone.

```bash
# Download the version matching the Velero shipped by your OADP release
curl -L -o velero.tar.gz \
  https://github.com/vmware-tanzu/velero/releases/download/v1.14.0/velero-v1.14.0-linux-amd64.tar.gz
tar -xzf velero.tar.gz
sudo mv velero-v1.14.0-linux-amd64/velero /usr/local/bin/
velero version -n openshift-adp
```

Then:

```bash
velero backup describe backup-test-001 -n openshift-adp --details
velero backup logs backup-test-001 -n openshift-adp
```

---

# Part 8 — Scheduled production backups

## 8.1 Naming and labeling convention

Agree on this before you create schedules. Suggested scheme:

- Schedule name: `<tier>-<namespace-or-group>-<frequency>` → `tier1-payments-6h`
- Every Schedule carries labels: `tier`, `owner`, `app`

## 8.2 Tier-1: frequent, with volume data in S3

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: tier1-payments-6h
  namespace: openshift-adp
  labels:
    tier: "1"
    owner: payments-team
spec:
  schedule: "0 */6 * * *"          # every 6 hours
  useOwnerReferencesInBackup: false
  template:
    includedNamespaces:
      - payments-prod
      - payments-db-prod
    snapshotMoveData: true
    storageLocation: default
    ttl: 720h0m0s                  # 30 days
    excludedResources:
      - events
      - events.events.k8s.io
      - pods                       # pods are recreated by their controllers
      - replicasets
```

## 8.3 Tier-2: daily, metadata plus volumes

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: tier2-apps-daily
  namespace: openshift-adp
  labels:
    tier: "2"
spec:
  schedule: "0 2 * * *"            # 02:00 every day
  template:
    includedNamespaces:
      - api-prod
      - web-prod
      - cache-prod
    defaultVolumesToFsBackup: true
    storageLocation: default
    ttl: 720h0m0s
    excludedResources:
      - events
      - events.events.k8s.io
```

## 8.4 Tier-3: weekly, metadata only

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: tier3-dev-weekly
  namespace: openshift-adp
  labels:
    tier: "3"
spec:
  schedule: "0 4 * * 0"            # 04:00 Sunday
  template:
    includedNamespaces:
      - team-a-dev
      - team-b-dev
    snapshotVolumes: false         # metadata only, no volume data
    storageLocation: default
    ttl: 336h0m0s                  # 14 days
```

## 8.5 Cluster-scoped configuration backup

This one protects the cluster-wide objects your apps depend on — the ones that are easy to forget until a rebuild.

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: cluster-scoped-config-daily
  namespace: openshift-adp
spec:
  schedule: "0 1 * * *"
  template:
    includedNamespaces:
      - "*"
    includedResources:
      - customresourcedefinitions
      - clusterroles.rbac.authorization.k8s.io
      - clusterrolebindings.rbac.authorization.k8s.io
      - securitycontextconstraints.security.openshift.io
      - storageclasses.storage.k8s.io
      - volumesnapshotclasses.snapshot.storage.k8s.io
      - priorityclasses.scheduling.k8s.io
      - ingressclasses.networking.k8s.io
    includeClusterResources: true
    snapshotVolumes: false
    storageLocation: default
    ttl: 720h0m0s
```

This is a **reference copy** for rebuild situations, not a one-click cluster restore. Treat the YAML in it as something you read and selectively re-apply, not something you blindly restore.

## 8.6 Using a label selector instead of listing namespaces

Listing namespaces by name goes stale. A more maintainable pattern is to label namespaces and select on the label:

```bash
oc label namespace payments-prod backup-tier=tier1
```

```yaml
spec:
  template:
    includedNamespaces:
      - "*"
    labelSelector:
      matchLabels:
        backup-tier: tier1
```

Note that the label selector applies to the **resources**, so you also need to label the objects, or use `orLabelSelectors`. A simpler and very common approach is to keep the explicit namespace list in Git and review it monthly.

## 8.7 Schedule spacing rules

- Leave enough time for a backup to finish before the next one starts. If a namespace takes 20 minutes, do not schedule every 15 minutes.
- Stagger schedules so they do not all fire at 02:00. Spread them across the window.
- Avoid overlapping with your cluster upgrade or patch windows.

## 8.8 Verify your schedules

```bash
oc get schedule -n openshift-adp
oc get backup -n openshift-adp --sort-by=.metadata.creationTimestamp
```

Schedules create Backups named `<schedule-name>-<timestamp>`.

To trigger a scheduled backup immediately (useful for testing):

```bash
velero backup create --from-schedule tier1-payments-6h -n openshift-adp
```

---

# Part 9 — Restore procedures

## 9.1 Pre-restore checklist — always do this first

Before any restore, in any scenario:

```bash
# 1. List available backups and confirm which one you want
oc get backup -n openshift-adp --sort-by=.metadata.creationTimestamp

# 2. Confirm that backup is Completed (not PartiallyFailed)
oc get backup <backup-name> -n openshift-adp -o jsonpath='{.status.phase}{"\n"}'

# 3. Look at what is actually in it
velero backup describe <backup-name> -n openshift-adp --details

# 4. Confirm the storage location is reachable right now
oc get backupstoragelocation -n openshift-adp

# 5. Tell the application owner and record the start time in your incident ticket
```

## 9.2 Scenario A — Restore a namespace over itself (same cluster)

The most common case: someone deleted or corrupted resources in a namespace.

**Velero will not overwrite existing resources by default.** It skips anything that already exists. So you have two options.

**Option 1 — delete the namespace first (cleanest):**

```bash
# WARNING: this destroys everything currently in the namespace
oc delete project payments-prod

# Wait for full termination — this can take several minutes
oc get project payments-prod
# repeat until it returns "not found"
```

Then restore:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore-payments-20260913
  namespace: openshift-adp
spec:
  backupName: tier1-payments-6h-20260913020000
  restorePVs: true
  excludedResources:
    - nodes
    - events
    - events.events.k8s.io
    - backups.velero.io
    - restores.velero.io
    - resticrepositories.velero.io
    - csinodes.storage.k8s.io
    - volumeattachments.storage.k8s.io
EOF
```

**Option 2 — restore into a live namespace and update existing objects:**

```yaml
spec:
  backupName: tier1-payments-6h-20260913020000
  existingResourcePolicy: update   # patch existing objects instead of skipping them
  restorePVs: true
```

Use `existingResourcePolicy: update` with care. It will overwrite live config with backup config.

## 9.3 Scenario B — Restore into a different namespace (safe testing)

The safest way to validate a backup without touching production:

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore-payments-validation
  namespace: openshift-adp
spec:
  backupName: tier1-payments-6h-20260913020000
  namespaceMapping:
    payments-prod: payments-restore-test
  restorePVs: true
```

After validation, delete `payments-restore-test`.

Note: Routes will collide if the restored Route hostname is identical. Either exclude Routes from the validation restore or accept that one will fail to admit.

## 9.4 Scenario C — Restore only specific resources

To restore just a ConfigMap or a Secret someone deleted:

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: restore-single-configmap
  namespace: openshift-adp
spec:
  backupName: tier1-payments-6h-20260913020000
  includedNamespaces:
    - payments-prod
  includedResources:
    - configmaps
  labelSelector:
    matchLabels:
      app: payments-api
  restorePVs: false
```

## 9.5 Scenario D — Restore to a different cluster (disaster recovery)

This is your real DR path. It only works if your backup data is in S3 (Data Mover or FSB), not CSI-snapshot-only.

**Steps:**

1. **Build the new cluster** from your install config. This is not something OADP does for you.
2. **Restore Layer 3 first:** apply your GitOps repo — operators, MachineConfigs, StorageClasses, network policy, IDP config. The target cluster must have the same StorageClass names as the source, or you must remap them (step 4).
3. **Install OADP on the new cluster** (Parts 2–5) pointing at the **same bucket and prefix** as the source cluster.
4. **Wait for backup discovery.** Velero syncs backup metadata from the bucket automatically — usually within a minute:
   ```bash
   oc get backup -n openshift-adp
   # you should see the source cluster's backups appear
   ```
5. **Map StorageClasses if names differ.** Create a ConfigMap in the OADP namespace:
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: change-storage-class-config
     namespace: openshift-adp
     labels:
       velero.io/plugin-config: ""
       velero.io/change-storage-class: RestoreItemAction
   data:
     old-storage-class-name: new-storage-class-name
   ```
6. **Restore namespace by namespace**, starting with the least critical, to validate the process:
   ```yaml
   apiVersion: velero.io/v1
   kind: Restore
   metadata:
     name: dr-restore-payments
     namespace: openshift-adp
   spec:
     backupName: tier1-payments-6h-20260913020000
     restorePVs: true
     existingResourcePolicy: update
   ```
7. **Fix cluster-specific values by hand.** Routes carry the old cluster's apps domain and will need updating. Check external DNS, load balancer targets, and any ConfigMap containing the old cluster hostname.

**Set expectations honestly:** a cross-cluster restore is never fully automatic. Budget hours, not minutes, and rehearse it (Part 12).

## 9.6 Monitor any restore

```bash
oc get restore <restore-name> -n openshift-adp -w
velero restore describe <restore-name> -n openshift-adp --details
velero restore logs <restore-name> -n openshift-adp
```

Restore phases: `New → InProgress → WaitingForPluginOperations → Completed` (or `PartiallyFailed` / `Failed`).

`PartiallyFailed` is common and often benign — for example, Velero could not recreate an object that the cluster manages itself. Read the warnings before you panic, but do not ignore them either.

## 9.7 Post-restore verification — always do these

```bash
# 1. All pods running
oc get pods -n payments-prod

# 2. PVCs bound
oc get pvc -n payments-prod

# 3. Actual data present — the real test
POD=$(oc get pod -n payments-prod -l app=payments -o name | head -1)
oc exec -n payments-prod $POD -- ls -la /data

# 4. Routes admitted and resolving
oc get route -n payments-prod

# 5. Secrets and ServiceAccounts present
oc get secret,sa -n payments-prod

# 6. Have the application owner run their own smoke test
```

## 9.8 DeploymentConfig cleanup (older clusters only)

If your applications still use `DeploymentConfig` objects and you restored with file-system backup, Velero scales DCs down during restore and leaves orphaned standalone pods behind. Run the Red Hat-provided cleanup script afterwards:

```bash
bash dc-restic-post-restore.sh <restore-name>
```

The script is published in the Red Hat OADP documentation. If your workloads use `Deployment` rather than `DeploymentConfig`, this does not apply.

---

# Part 10 — etcd / control-plane backup and restore

OADP does not protect etcd. This is a separate, mandatory process.

## 10.1 What an etcd backup contains

A full snapshot of every object in the cluster API: all namespaces, all secrets, RBAC, operator config, cluster operator state. It is your only recovery path for a control-plane failure or a lost-quorum event.

**It also contains every Secret in the cluster in readable form.** Treat the snapshot file as a top-tier sensitive artifact: encrypt it at rest and restrict access to it more tightly than your application backups.

## 10.2 Take an etcd backup

Run this on **one** control plane node. You only need it from one node — etcd is replicated.

```bash
# List control plane nodes
oc get nodes -l node-role.kubernetes.io/master

# Open a debug session on one of them
oc debug node/<master-node-name>

# Inside the debug pod:
chroot /host
sudo -E /usr/local/bin/cluster-backup.sh /home/core/assets/backup
```

The script produces two files in that directory:

| File | Contents |
|---|---|
| `snapshot_<datetime>.db` | The etcd database snapshot |
| `static_kuberesources_<datetime>.tar.gz` | Static pod manifests and the etcd encryption keys |

You need **both** files to restore. Never separate them.

## 10.3 Copy the backup off the node and into S3

The backup is useless if it stays on a node that dies with the cluster.

```bash
# From your workstation, copy the files out of the debug pod's node filesystem
oc debug node/<master-node-name> -- \
  chroot /host tar czf - /home/core/assets/backup > etcd-backup-$(date +%Y%m%d).tar.gz

# Encrypt before uploading
gpg --symmetric --cipher-algo AES256 etcd-backup-$(date +%Y%m%d).tar.gz

# Upload to S3, ideally to a separate bucket with tighter access control
aws s3 cp etcd-backup-$(date +%Y%m%d).tar.gz.gpg \
  s3://ocp-etcd-backups/cluster-dfw-prod-01/ \
  --endpoint-url https://s3.internal.example.com

# Shred the local copies
shred -u etcd-backup-*.tar.gz etcd-backup-*.tar.gz.gpg
```

## 10.4 Automate it

Automate the snapshot with a CronJob or your existing scheduler, and always take a manual one immediately before:

- Any cluster upgrade
- Any certificate rotation
- Any change to the identity provider or cluster-wide RBAC
- Any operator install or major config change

Keep at least the last 7–14 daily snapshots. Old etcd snapshots become less useful over time because certificates expire; a snapshot older than your certificate rotation period may not restore cleanly.

## 10.5 Verify a snapshot is readable

```bash
oc debug node/<master-node-name> -- chroot /host \
  ls -lh /home/core/assets/backup/
# confirm the .db file is non-zero and roughly the size you expect (typically 100 MB – several GB)
```

## 10.6 Restoring etcd — read this before you act

**etcd restore is a destructive, last-resort operation.** It reverts the entire cluster to the state at snapshot time. Everything created after that point is gone: new namespaces, new secrets, new certificates.

**Before you restore etcd, confirm you actually need to.** Most incidents that look like "we need to restore etcd" are actually:

- A single failed control plane node → replace the node, do not restore etcd
- A lost etcd member → remove and re-add the member, do not restore etcd
- A deleted namespace → restore it with OADP, do not restore etcd

etcd restore is appropriate when: the majority of control plane nodes are lost, or a cluster-wide change corrupted the API state.

**When you do restore:**

1. **Open a Red Hat support case first** if you have a subscription. This procedure is version-specific and the docs for your exact OpenShift version are authoritative.
2. Follow the "Restoring to a previous cluster state" procedure in the Backup and Restore guide for **your exact OpenShift version**. The procedure changed meaningfully across 4.x releases (notably the introduction of the quorum-restore path).
3. The broad shape of the procedure:
   - Choose a healthy control plane node as the recovery host.
   - Copy both backup files to that node.
   - Stop the static pods on all control plane nodes.
   - Run `/usr/local/bin/cluster-restore.sh /home/core/assets/backup` on the recovery host.
   - Restart kubelet on the recovery host, verify the single-member etcd comes up.
   - Bring the remaining control plane nodes back, forcing them to rejoin.
   - Verify all cluster operators return to Available.
   - Rotate certificates and signer keys as the documented procedure directs.
4. Expect several hours and expect worker nodes to need attention afterwards.

Because the exact commands vary by version, this runbook deliberately does not reproduce them. Look them up fresh, for your version, at restore time.

---

# Part 11 — Monitoring and alerting

A backup system nobody watches is a backup system that has been failing silently for three months.

## 11.1 Daily health check

```bash
#!/bin/bash
# oadp-health-check.sh — run daily, alert on non-zero exit

NS=openshift-adp
FAIL=0

echo "=== Backup Storage Location ==="
PHASE=$(oc get backupstoragelocation default -n $NS -o jsonpath='{.status.phase}')
echo "BSL phase: $PHASE"
[ "$PHASE" != "Available" ] && { echo "ALERT: BSL not Available"; FAIL=1; }

echo "=== Backups in the last 24h ==="
oc get backup -n $NS -o json | jq -r '
  .items[]
  | select(.metadata.creationTimestamp > (now-86400 | todate))
  | "\(.metadata.name)\t\(.status.phase)"'

echo "=== Failed backups in the last 7 days ==="
BAD=$(oc get backup -n $NS -o json | jq -r '
  .items[]
  | select(.metadata.creationTimestamp > (now-604800 | todate))
  | select(.status.phase != "Completed")
  | .metadata.name')
[ -n "$BAD" ] && { echo "ALERT: failed/partial backups: $BAD"; FAIL=1; }

echo "=== Node agent pods ==="
NODES=$(oc get nodes --no-headers | wc -l)
AGENTS=$(oc get pods -n $NS -l name=node-agent --field-selector=status.phase=Running --no-headers | wc -l)
echo "node-agent pods running: $AGENTS / nodes: $NODES"

exit $FAIL
```

## 11.2 Prometheus alerts

Velero exposes metrics. Create a PrometheusRule (requires user-workload monitoring enabled):

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: oadp-backup-alerts
  namespace: openshift-adp
spec:
  groups:
    - name: oadp.rules
      rules:
        - alert: VeleroBackupFailure
          expr: increase(velero_backup_failure_total[1h]) > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "A Velero backup failed in the last hour"

        - alert: VeleroBackupPartialFailure
          expr: increase(velero_backup_partial_failure_total[1h]) > 0
          for: 5m
          labels:
            severity: warning
            annotations:
              summary: "A Velero backup partially failed"

        - alert: VeleroNoRecentBackup
          expr: time() - velero_backup_last_successful_timestamp > 86400
          for: 30m
          labels:
            severity: critical
          annotations:
            summary: "No successful backup for schedule {{ $labels.schedule }} in over 24 hours"

        - alert: VeleroBSLUnavailable
          expr: velero_backup_storage_location_availability != 1
          for: 15m
          labels:
            severity: critical
          annotations:
            summary: "Velero backup storage location is not available"
```

The `VeleroNoRecentBackup` alert is the most valuable one here — it catches the silent failure mode where schedules simply stop firing.

## 11.3 Monthly review items

- Bucket size and growth trend versus your quota
- Whether any Schedule is producing zero backups
- Whether new namespaces have been created that no Schedule covers — this is the most common gap
- Whether `ttl` values still match your retention policy

---

# Part 12 — DR drill checklist

Run this **quarterly**. A backup you have never restored is a hypothesis, not a backup.

| # | Step | Pass criteria | Who |
|---|---|---|---|
| 1 | Pick a random production backup from the last 7 days | Backup phase is `Completed` | Engineer |
| 2 | Restore it into a new namespace using `namespaceMapping` | Restore reaches `Completed` | Engineer |
| 3 | Verify pod count matches the source | Same number of running pods | Engineer |
| 4 | Verify PVC data is present and current | App owner confirms data | App owner |
| 5 | Time the whole operation | Within your documented RTO | Engineer |
| 6 | Restore one namespace to the DR cluster | Restore completes; app starts | Engineer |
| 7 | Verify the etcd snapshot from last night is downloadable and decryptable | File opens, size is plausible | Engineer |
| 8 | Confirm every production namespace is covered by a Schedule | Zero uncovered namespaces | Engineer |
| 9 | Record results and update this runbook | Document committed to Git | Engineer |
| 10 | Clean up the test namespaces | No leftover restore artifacts | Engineer |

Log each drill with date, who ran it, elapsed time, and what broke. The "what broke" column is the most useful part of the record.

---

# Part 13 — Troubleshooting

## 13.1 BackupStorageLocation is `Unavailable`

| Symptom in logs | Cause | Fix |
|---|---|---|
| `NoCredentialProviders` | Secret missing or wrong key name | Secret must be named as referenced in the DPA, with key `cloud` |
| `InvalidAccessKeyId` / `SignatureDoesNotMatch` | Wrong keys, or whitespace/quotes in the credentials file | Recreate the Secret; check for trailing spaces |
| `certificate signed by unknown authority` | Self-signed S3 cert | Add `caCert` (base64, single line) to the DPA — Section 4.2 |
| `dial tcp ... i/o timeout` | Firewall or proxy blocking nodes | Test with `oc debug node/... -- chroot /host curl <s3url>` |
| `NoSuchBucket` | Bucket does not exist or wrong region | Verify bucket name and endpoint |
| `301 Moved Permanently` | Path-style addressing needed | Set `s3ForcePathStyle: "true"` |

## 13.2 Backup stuck `InProgress`

```bash
oc logs -n openshift-adp deployment/velero --tail=100
oc get datauploads -n openshift-adp        # Data Mover uploads
oc get podvolumebackups -n openshift-adp   # File System Backup
oc logs -n openshift-adp -l name=node-agent --tail=100
```

Common causes:
- The volume is very large and the upload is genuinely still running — check `.status.progress`
- A node agent pod is not running on the node hosting the PVC
- A pre-backup hook is hanging — check the hook's `timeout`

## 13.3 Backup `PartiallyFailed`

```bash
velero backup describe <name> -n openshift-adp --details
velero backup logs <name> -n openshift-adp | grep -i error
```

Frequently harmless: failures to back up `events`, terminating pods, or objects managed by controllers. Frequently serious: any PVC or Secret that failed. Read every error; do not accept `PartiallyFailed` as normal for a Tier-1 backup.

## 13.4 Restore leaves pods in `Pending`

| Cause | Check | Fix |
|---|---|---|
| PVC cannot bind | `oc get pvc -n <ns>` | StorageClass missing on target cluster — use the storage class mapping ConfigMap (Section 9.5) |
| Node selector / taint mismatch | `oc describe pod` | Target cluster has different node labels |
| Image pull failure | `oc describe pod` | Registry not reachable or pull secret not restored |
| SCC denial | `oc describe pod`, check events | ServiceAccount's SCC binding was not restored |

## 13.5 Restored data is empty

The most likely cause is that the backup used CSI snapshots only, and you restored to a different cluster or after the snapshots expired. Check:

```bash
velero backup describe <name> -n openshift-adp --details | grep -A5 -i "volume"
```

If it shows CSI snapshots with no data upload, the data never left your storage array. Change those schedules to use `snapshotMoveData: true`.

## 13.6 Gather logs for a support case

```bash
oc adm must-gather --image=registry.redhat.io/oadp/oadp-mustgather-rhel9:latest
```

---

# Part 14 — Command cheat sheet

```bash
# --- Status ---
oc get dpa -n openshift-adp
oc get backupstoragelocation -n openshift-adp
oc get pods -n openshift-adp
oc get backup,restore,schedule -n openshift-adp

# --- Backups ---
oc get backup -n openshift-adp --sort-by=.metadata.creationTimestamp
velero backup describe <name> -n openshift-adp --details
velero backup logs <name> -n openshift-adp
velero backup create adhoc-$(date +%Y%m%d%H%M) --include-namespaces myapp -n openshift-adp
velero backup create --from-schedule <schedule-name> -n openshift-adp
velero backup delete <name> -n openshift-adp

# --- Restores ---
velero restore create --from-backup <backup> -n openshift-adp
velero restore describe <name> -n openshift-adp --details
velero restore logs <name> -n openshift-adp

# --- Schedules ---
oc get schedule -n openshift-adp
velero schedule pause <name> -n openshift-adp
velero schedule unpause <name> -n openshift-adp

# --- Volume operations ---
oc get datauploads -n openshift-adp        # Data Mover backup
oc get datadownloads -n openshift-adp      # Data Mover restore
oc get podvolumebackups -n openshift-adp   # File System Backup
oc get volumesnapshots -A

# --- Logs ---
oc logs -n openshift-adp deployment/velero -f
oc logs -n openshift-adp -l name=node-agent --tail=100

# --- etcd ---
oc debug node/<master> -- chroot /host /usr/local/bin/cluster-backup.sh /home/core/assets/backup
oc get etcd -o=jsonpath='{range .items[0].status.conditions[?(@.type=="EtcdMembersAvailable")]}{.message}{"\n"}'
```

---

# Appendix A — Rollout plan

Do not go from zero to full production coverage in one day.

| Week | Activity | Exit criteria |
|---|---|---|
| 1 | Prerequisites, S3 bucket, IAM, network validation | `curl` to S3 succeeds from a worker node |
| 1 | Install OADP, create Secret and DPA | BSL phase `Available` |
| 2 | Test backup and restore in `backup-test` namespace | Proof file recovered intact |
| 2 | Snapshot class validation, choose volume method per tier | Decision documented per app |
| 3 | Onboard Tier-3 (dev) namespaces, run for a week | Backups complete nightly |
| 4 | Onboard Tier-2, set up monitoring and alerts | Alerts fire on a deliberately broken backup |
| 5 | Onboard Tier-1 with hooks, validate app-consistent restore | App owner signs off |
| 6 | etcd backup automation, first full DR drill | Drill results documented |

---

# Appendix B — RBAC for the team

Cluster-admin is needed to install OADP. Day-to-day backup operations do not require it. Create a scoped role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: oadp-operator-user
rules:
  - apiGroups: ["velero.io"]
    resources: ["backups", "restores", "schedules", "backupstoragelocations", "downloadrequests"]
    verbs: ["get", "list", "watch", "create"]
  - apiGroups: ["oadp.openshift.io"]
    resources: ["dataprotectionapplications"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list"]
```

Note that `create` on Restore is effectively a powerful permission — a restore can overwrite live resources. Grant deliberately, and require a change ticket for any production restore.

---

# Appendix C — Quick reference: which field do I set?

| I want to... | Field |
|---|---|
| Back up specific namespaces | `includedNamespaces: [a, b]` |
| Exclude noisy objects | `excludedResources: [events, pods]` |
| Copy volume data into S3 via snapshot | `snapshotMoveData: true` |
| Copy volume data into S3 via file copy | `defaultVolumesToFsBackup: true` |
| Back up metadata only, no volumes | `snapshotVolumes: false` |
| Set retention | `ttl: 720h0m0s` |
| Restore into a different namespace | `namespaceMapping: {old: new}` |
| Overwrite existing objects on restore | `existingResourcePolicy: update` |
| Skip volume restore | `restorePVs: false` |
| Run a command before/after backup | `hooks.resources[].pre` / `.post` |

---

**Document owner:** _______________
**Last reviewed:** _______________
**Next DR drill due:** _______________