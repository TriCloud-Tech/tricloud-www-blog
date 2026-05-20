---
title: OpenShift Virtualization (KubeVirt) in Airgapped Environments
date: 2026-05-20
excerpt: A deep-dive into running OpenShift Virtualization in fully disconnected networks — covering architecture, image mirroring with oc-mirror, the HyperConverged Operator, and the new aba toolchain.
image: https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80
---

# OpenShift Virtualization (KubeVirt) in Airgapped Environments

Many regulated industries — defence, financial services, critical national infrastructure — mandate that workloads run on networks that have **zero egress to the public internet**. Running OpenShift Virtualization (the productised form of KubeVirt) in these environments requires extra planning: every container image, every operator catalogue, and every boot source must be pre-staged inside the enclave before a single VM can start.

This post walks through the full picture: what OpenShift Virtualization actually is, how its components are structured, and exactly what you need to do to get it running when `registry.redhat.io` is simply unreachable.

---

## What is OpenShift Virtualization?

OpenShift Virtualization is Red Hat's productised distribution of [KubeVirt](https://kubevirt.io), an upstream CNCF project that adds VM lifecycle management as a first-class Kubernetes primitive. It uses Linux KVM under the hood, exposing VMs as `VirtualMachine` custom resources inside an OCP cluster. The result is a single control plane for both container and VM workloads, enabling teams to migrate legacy VM-based applications at their own pace without needing a separate virtualisation platform.

IBM ships and supports OpenShift Virtualization as part of its **Red Hat OpenShift Virtualization Service on IBM Cloud** offering, targeting customers who want to consolidate VM and container estates onto a single, Kubernetes-native platform.

> "OpenShift Virtualization uses KubeVirt and KVM to allow teams to migrate and operate VM-based applications directly inside Red Hat OpenShift."  
> — [IBM Cloud, Red Hat OpenShift Virtualization Service](https://www.ibm.com/products/openshift-virtualization)

---

## Architecture Overview

OpenShift Virtualization is not a single operator — it is a layered stack of operators co-ordinated by a single meta-operator called the **HyperConverged Operator (HCO)**.

```
┌──────────────────────────────────────────────────────────────────┐
│                  OpenShift Container Platform                    │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           HyperConverged Operator  (HCO)                 │   │
│  │                                                          │   │
│  │   ┌────────────┐  ┌─────────────┐  ┌─────────────────┐  │   │
│  │   │  KubeVirt  │  │  CDI (Cont. │  │  Cluster Network│  │   │
│  │   │  Operator  │  │  Data Impr) │  │  Addons (CNA)   │  │   │
│  │   └─────┬──────┘  └──────┬──────┘  └────────┬────────┘  │   │
│  │         │                │                  │            │   │
│  │   ┌─────▼──────┐  ┌──────▼──────┐  ┌────────▼────────┐  │   │
│  │   │virt-api    │  │ cdi-apisvr  │  │ multus-cni      │  │   │
│  │   │virt-ctrl   │  │ cdi-ctrl    │  │ linux-bridge-cni│  │   │
│  │   │virt-handler│  │ cdi-deployer│  │ ovs-cni         │  │   │
│  │   │virt-operator│ └─────────────┘  └─────────────────┘  │   │
│  │   └────────────┘                                         │   │
│  │                                                          │   │
│  │   ┌─────────────────────┐  ┌──────────────────────────┐  │   │
│  │   │ SSP Operator        │  │ Hostpath Provisioner     │  │   │
│  │   │ (scheduling, scale) │  │ Operator (HPP)           │  │   │
│  │   └─────────────────────┘  └──────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Worker Nodes                                            │   │
│  │  ┌──────────────┐  ┌────────────────────────────────┐   │   │
│  │  │ virt-handler │  │  QEMU / libvirt / KVM          │   │   │
│  │  │  (DaemonSet) │  │  (per-node hypervisor layer)   │   │   │
│  │  └──────────────┘  └────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### Component Breakdown

| Component | Role |
|---|---|
| **HyperConverged Operator (HCO)** | Single entry-point meta-operator; creates CRs for all sub-operators and enforces opinionated defaults |
| **virt-operator** | Manages KubeVirt lifecycle; reconciles the `KubeVirt` CR |
| **virt-api** | Kubernetes API extension; handles VM CRD operations |
| **virt-controller** | Control-plane daemon; schedules and manages `VirtualMachineInstance` pods |
| **virt-handler** | Node-level DaemonSet; interfaces with QEMU/libvirt on each worker |
| **CDI (Containerized Data Importer)** | Imports VM disk images into PVCs via `DataVolume` CRs |
| **CNA (Cluster Network Addons)** | Deploys Multus, Linux Bridge CNI, OVS CNI for VM networking |
| **SSP Operator** | Manages scheduling scale and performance policies for VMs |
| **HPP Operator** | Provides hostpath-based persistent storage for VM disks |

> Source: [Red Hat OpenShift Virtualization Architecture, OCP 4.12](https://docs.redhat.com/en/documentation/openshift_container_platform/4.12/html/virtualization/virt-architecture)

---

## The Airgap Challenge

In a standard connected deployment, the Operator Lifecycle Manager (OLM) reaches out to `registry.redhat.io` and `quay.io` to:

1. Pull the operator catalogue index image
2. Resolve the operator bundle
3. Pull component container images on first install and on upgrades

In an airgapped environment **none of these calls can succeed**. You must:

- Mirror all required images to a private registry inside the enclave
- Replace the default `CatalogSource` with one pointing at your mirror
- Apply an `ImageContentSourcePolicy` (or `ImageDigestMirrorSet` in OCP 4.13+) so every image pull is transparently redirected

```
┌───────────────────────┐          ┌──────────────────────────────┐
│   CONNECTED ZONE      │          │   AIRGAPPED ENCLAVE          │
│                       │          │                              │
│  registry.redhat.io ──┼──────────┼──▶  Mirror Registry         │
│  quay.io              │  (one-   │     (Quay, Harbor, Nexus…)   │
│  Red Hat CDN          │   time   │                              │
│                       │  copy)   │  ┌────────────────────────┐  │
│                       │          │  │  OpenShift Cluster     │  │
│                       │          │  │                        │  │
│                       │          │  │  OLM ──▶ CatalogSource │  │
│                       │          │  │          (mirrored)    │  │
│                       │          │  │                        │  │
│                       │          │  │  ICSP / IDMS redirects │  │
│                       │          │  │  all pulls to mirror   │  │
│                       │          │  └────────────────────────┘  │
└───────────────────────┘          └──────────────────────────────┘
```

---

## Three Deployment Scenarios

### Scenario 1 — Partially Connected (Proxy)

The cluster has outbound internet access via an HTTP/S proxy. OCP and OLM can be configured with `proxy` settings; no image mirroring is strictly required, though mirroring still reduces egress and improves pull performance.

### Scenario 2 — Fully Disconnected (Mirror-to-Registry)

A bastion host with internet access mirrors images to a portable disk (tar file). The disk is physically moved into the enclave and loaded into the internal mirror registry. This is the most common scenario for government and defence deployments.

### Scenario 3 — Developer Preview — ISO-Embedded (OCP 4.19+)

Red Hat introduced a new **Agent ISO** approach in OCP 4.19 (Developer Preview). A single ~40 GB ISO is pre-loaded with the full OCP 4.19 release payload **plus** the virtualization operators. No pre-existing internal registry is required for initial bootstrap.

> "Seamless deployment in air-gapped networks without a pre-existing image registry."  
> — [Red Hat Developer, Disconnected OpenShift Virtualization Made Easy, August 2025](https://developers.redhat.com/articles/2025/08/15/disconnected-openshift-virtualization-made-easy)

---

## Method 1: Mirror-to-Registry with `oc-mirror`

This is the **production-recommended approach** for OCP 4.11–4.19 clusters in fully disconnected environments.

### Prerequisites

- A RHEL 8/9 bastion host with internet access (used once for mirroring)
- A private container registry inside the enclave (e.g. Red Hat Quay, Harbor, Nexus Repository)
- OpenShift CLI (`oc`) + `oc-mirror` plugin installed on the bastion
- A valid Red Hat pull secret from [console.redhat.com](https://console.redhat.com)

### Step 1 — Install `oc-mirror`

```bash
tar xvzf oc-mirror.tar.gz
chmod +x oc-mirror
sudo mv oc-mirror /usr/local/bin/

# Verify
oc mirror help
```

### Step 2 — Configure credentials

```bash
# Place your pull secret where podman/skopeo can find it
mkdir -p ~/.docker
cat ./pull-secret | jq . > ~/.docker/config.json

# Verify login
podman login registry.redhat.io
```

### Step 3 — Discover the OpenShift Virtualization operator

```bash
# List available operator packages in the Red Hat catalog
oc mirror list operators \
  --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.17 \
  --package=kubevirt-hyperconverged

# Find available channels and versions
oc mirror list operators \
  --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.17 \
  --package=kubevirt-hyperconverged \
  --channel=stable
```

### Step 4 — Create an `ImageSetConfiguration`

```yaml
# virt-mirror-config.yaml
kind: ImageSetConfiguration
apiVersion: mirror.openshift.io/v1alpha2
storageConfig:
  local:
    path: /mnt/mirror-metadata
mirror:
  platform:
    channels:
    - name: stable-4.17
      type: ocp
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.17
    targetCatalog: redhat-catalog-v4.17
    packages:
    - name: kubevirt-hyperconverged      # OpenShift Virtualization / HCO
      channels:
      - name: stable
    - name: local-storage-operator       # Required for VM disk storage
      channels:
      - name: stable
    - name: odf-operator                 # Optional — OpenShift Data Foundation
      channels:
      - name: stable-4.17
  additionalImages:
  # Common VM boot sources (Fedora, RHEL, Windows)
  - name: registry.redhat.io/rhel8/rhel-guest-image:latest
  - name: registry.redhat.io/rhel9/rhel-guest-image:latest
```

### Step 5 — Mirror to disk (on bastion, internet-connected)

```bash
mkdir /mnt/virt-mirror-disk

oc mirror --verbose 3 \
  --config=virt-mirror-config.yaml \
  file:///mnt/virt-mirror-disk
```

This produces a tar archive (`mirror_seq1_000000.tar`) containing all layers and manifests. Transfer this to your enclave via approved media (USB, courier disk, data diode).

### Step 6 — Load images into the internal registry (inside enclave)

```bash
# On a host inside the enclave with access to the mirror registry
oc mirror --verbose 3 \
  --from=./mirror_seq1_000000.tar \
  docker://registry.internal.example.com:5000/openshift4
```

`oc-mirror` writes its generated manifests to `oc-mirror-workspace/results-*/`:

```
oc-mirror-workspace/
└── results-1234567890/
    ├── imageContentSourcePolicy.yaml      # Image redirect rules
    ├── catalogSource-redhat-catalog-v4-17.yaml
    └── mapping.txt
```

### Step 7 — Apply cluster-wide image redirect rules

```bash
# Disable the default (internet-facing) OperatorHub sources
oc patch OperatorHub cluster --type json \
  -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'

# Apply the ImageContentSourcePolicy (ICSP)
oc apply -f oc-mirror-workspace/results-*/imageContentSourcePolicy.yaml

# Register the mirrored catalog
oc apply -f oc-mirror-workspace/results-*/catalogSource-redhat-catalog-v4-17.yaml
```

After applying the ICSP, the cluster-wide `MachineConfigPool` will roll out an update to all nodes — **this causes a rolling reboot**. Plan for this maintenance window.

### Step 8 — Install the HyperConverged Operator via OLM

Once nodes are back and the mirrored `CatalogSource` is healthy:

```bash
# Create the target namespace
oc create namespace openshift-cnv

# Create the OperatorGroup
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: kubevirt-hyperconverged-group
  namespace: openshift-cnv
spec:
  targetNamespaces:
  - openshift-cnv
EOF

# Create the Subscription
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: hco-operatorhub
  namespace: openshift-cnv
spec:
  source: redhat-catalog-v4-17          # Your mirrored catalog name
  sourceNamespace: openshift-marketplace
  name: kubevirt-hyperconverged
  startingCSV: kubevirt-hyperconverged-operator.v4.17.0
  channel: stable
EOF
```

### Step 9 — Create the HyperConverged CR

```bash
cat <<EOF | oc apply -f -
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec:
  # Disable boot source auto-update — not useful in airgapped environments
  # where no external image sources are reachable
  featureGates:
    enableCommonBootImageImport: false
EOF
```

Verify all HCO components become ready:

```bash
oc get csv -n openshift-cnv
oc get hyperconverged kubevirt-hyperconverged -n openshift-cnv -o jsonpath='{.status.conditions}' | jq .
```

---

## Method 2: `aba` Automation Tool (OCP 4.19+)

For teams deploying repeatedly or who want a more opinionated, tested workflow, Red Hat has released **aba** — an open-source CLI that wraps `oc-mirror` v2, the Agent-Based Installer, and Mirror Registry for Red Hat OpenShift into a single automation layer.

> "Aba packages everything you need in one place — install images, CLI tools, registry setup, and automation — enabling repeatable, tested workflows across disconnected deployments."  
> — [Red Hat Developer, Simplify OpenShift Installation in Air-Gapped Environments, October 2025](https://developers.redhat.com/articles/2025/10/14/simplify-openshift-installation-air-gapped-environments)

### Workflow (Fully Disconnected — 3 Steps)

```
┌──────────────────────────────────────────────────────────────────┐
│  STEP 1 — Bastion Preparation (internet-connected)              │
│                                                                  │
│  Download install bundle:  4.19.12-ocpv                         │
│  (includes OCP + ODF + OpenShift Virtualization operators)      │
│                                                                  │
│  Install aba + deps:                                             │
│  jq make python3-jinja2 podman skopeo openssl coreos-installer  │
│                                                                  │
│  ──────────────────────────────────────────────────────────────  │
│  STEP 2 — Mirror Registry Setup (inside enclave)                │
│                                                                  │
│  aba load \                                                      │
│    --mirror-hostname registry.example.com \                     │
│    --retry                                                       │
│                                                                  │
│  → Installs mirror registry                                     │
│  → Loads all images from bundle (10–20 min)                     │
│  → Configures auth                                              │
│                                                                  │
│  ──────────────────────────────────────────────────────────────  │
│  STEP 3 — Cluster Install                                        │
│                                                                  │
│  aba cluster --name prod-virt \                                  │
│              --type standard \                                   │
│              --starting-ip 10.0.1.100                           │
│                                                                  │
│  aba agentconf   → Generates install-config.yaml                │
│  aba iso         → Creates bootable Agent ISO                   │
│  aba mon         → Monitors installation progress               │
└──────────────────────────────────────────────────────────────────┘
```

**aba** supports:
- SNO (Single Node OpenShift), compact (3-node), and standard (3 control + N worker) topologies
- VLAN tagging and NIC bonding
- oc-mirror v2
- Multiple operator catalogue sources (Red Hat, Certified, Marketplace, Community)
- Arm64 (tested)

---

## IBM MAS and Disconnected OpenShift

IBM Maximo Application Suite (MAS) is commonly deployed on OpenShift in air-gapped industrial environments. IBM's documentation for [Installing MAS on a disconnected Red Hat OpenShift cluster](https://www.ibm.com/docs/en/mas-cd?topic=setup-installing-disconnected-red-hat-openshift-cluster) follows the same core pattern:

1. Mirror images from public registries to a private registry using `oc-mirror` or `oc adm catalog mirror`
2. Configure `ImageContentSourcePolicy` to redirect pulls
3. Disable default `OperatorHub` sources
4. Register mirrored catalogues as custom `CatalogSource` objects

IBM's guidance also emphasises using **Red Hat Marketplace** for operator delivery, which supports a dedicated disconnected operator install flow available at the [Red Hat Marketplace disconnected cluster documentation](https://swc.saas.ibm.com/en-us/redhat-marketplace/documentation/deploy-products-to-a-disconnected-environment).

---

## VM Boot Sources in Airgapped Environments

By default, OpenShift Virtualization populates `DataSource` objects that reference common OS boot images hosted on Red Hat's CDN. In a disconnected environment these references will fail unless you either:

**Option A — Disable automatic boot source imports (recommended for airgap):**

```yaml
# In the HyperConverged CR spec
spec:
  featureGates:
    enableCommonBootImageImport: false
```

**Option B — Pre-stage boot images as PVCs and create custom `DataSource` CRs:**

```yaml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: rhel9-base
  namespace: openshift-virtualization-os-images
spec:
  source:
    registry:
      url: "docker://registry.internal.example.com:5000/rhel9/rhel-guest-image:latest"
      pullMethod: node
  pvc:
    accessModes:
    - ReadWriteMany
    resources:
      requests:
        storage: 30Gi
```

---

## Network Considerations

OpenShift Virtualization VMs need layer-2 connectivity for many traditional workloads. Key networking components (all mirrored via the CNA operator) include:

```
┌──────────────────────────────────────────────────────────┐
│  VM Networking Stack (CNA-managed)                       │
│                                                          │
│  ┌──────────┐    ┌──────────────┐    ┌────────────────┐  │
│  │  VM Pod  │    │  Multus CNI  │    │  SDN / OVN-K   │  │
│  │          │───▶│  (multi-nic) │───▶│  (pod network) │  │
│  │  QEMU    │    └──────────────┘    └────────────────┘  │
│  └──────────┘           │                                │
│                         ▼                                │
│               ┌──────────────────┐                       │
│               │ Linux Bridge CNI │  ← trunk/access VLAN  │
│               │ or OVS CNI       │    to physical switch  │
│               └──────────────────┘                       │
└──────────────────────────────────────────────────────────┘
```

For VM live migration between nodes, ensure the cluster has a dedicated migration network with sufficient bandwidth (recommended: ≥10 GbE).

---

## Upgrade Considerations in Airgapped Environments

Upgrading OpenShift Virtualization in a disconnected cluster follows the same pattern as initial install:

1. Re-run `oc-mirror` (or `aba load`) with updated version ranges in the `ImageSetConfiguration`
2. Generate and apply a fresh `ImageContentSourcePolicy` (incremental diffs are supported in oc-mirror v2)
3. Update the `Subscription` `startingCSV` or allow OLM to auto-update within the approved channel
4. The HCO reconciles sub-operator upgrades sequentially to minimise disruption

For clusters managed by the **OpenShift Update Service (OSUS)**, `aba` can also configure a local OSUS instance inside the enclave, providing a graph-based update server identical to `api.openshift.com/api/upgrades_info` but served internally.

---

## Troubleshooting Common Airgap Issues

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `ImagePullBackOff` on virt-operator pods | ICSP not yet applied / node reboot pending | Check `oc get mcp` — wait for roll-out to complete |
| `CatalogSource` stuck in `CONNECTING` | Mirror registry TLS cert not trusted | Add CA cert to cluster trust via `oc create configmap` in `openshift-config`, patch `image.config.openshift.io` |
| CDI importer pod fails with `401 Unauthorized` | Pull secret missing for internal registry | Create `dockerconfigjson` secret in `openshift-virtualization-os-images` namespace |
| Boot source PVC stuck in `Pending` | StorageClass default not set | `oc patch storageclass <name> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'` |
| HCO reports `ReconcileError` | Sub-operator image tag not found in mirror | Re-run oc-mirror with broader `minVersion`/`maxVersion` range to include the missing bundle |

---

---

## Nice To Knows

These are the things that don't make it into the official docs but will save you real time on a project.

### 1. The Proxy Whitelist Keeps Changing — and It's Not Just Red Hat Domains

If you're in a partially-connected environment and routing outbound traffic through a proxy rather than mirroring everything, you'll quickly discover that the list of hostnames you need to allow is longer than you'd expect — and it shifts between OCP releases.

Red Hat publishes the [official list of required endpoints](https://docs.openshift.com/container-platform/4.17/installing/install_config/configuring-firewall.html), but in practice projects have been caught out by entries like these that aren't always obvious at first glance:

| Hostname | Why it's needed |
|---|---|
| `registry.redhat.io` | Operator and component images |
| `quay.io` | Some operator images are hosted here, not on registry.redhat.io |
| `cdn.quay.io` | Blob/layer storage backing quay.io pulls |
| `cdn01.quay.io`, `cdn02.quay.io`, `cdn03.quay.io` | CDN shards — all needed, not just the apex |
| `sso.redhat.io` | Authentication for registry.redhat.io |
| `api.openshift.com` | Cluster telemetry, update graph (OSUS) |
| `cert-api.access.redhat.com` | Certificate/entitlement checks |
| `access.redhat.com` | Subscription Manager |
| `infogw.api.openshift.com` | Insights operator |
| `console.redhat.com` | Pull secret validation, Assisted Installer |
| `rhcos.mirror.openshift.com` | RHCOS image downloads during install |
| `mirror.openshift.com` | OCP release artefacts |
| `storage.googleapis.com` | Some RHCOS artefacts are served from GCS |

The **GCS bucket (`storage.googleapis.com`)** is the one that most commonly surprises people — it's a Google domain and will be blocked by default in most corporate proxy policies. Raise it early with your network/security team. Similarly, `cdn01–03.quay.io` are often missed because engineers test against `quay.io` directly and assume that's sufficient.

> **Practical tip:** Run `oc adm must-gather` after your first install attempt and grep the logs for `connection refused` or `dial tcp` errors. This will surface any missing proxy entries far faster than auditing the docs manually.

If you're fully airgapped (no proxy at all), this is less relevant — but keep the list handy for the day someone asks you to justify why 14 external hostnames were in the change request.

---

### 2. Coming from VMware? Tell Your AI Assistant That Up Front

OpenShift Virtualization is a fundamentally different model from vSphere, but the *concepts* map reasonably well once you have the right translation layer. If you're using an AI coding assistant or chat tool to help you work through configuration problems, **start every conversation with your background**. Something like:

> "I have 10 years of VMware vSphere/NSX-T experience. I'm new to OpenShift Virtualization / KubeVirt. Please explain concepts by comparing them to VMware equivalents where possible."

This single prompt change dramatically improves the quality of explanations. For example, an AI that knows your background will naturally explain things like:

| VMware concept | OpenShift Virtualization equivalent |
|---|---|
| vCenter | OCP Console + `virtctl` CLI |
| ESXi host | Worker node running `virt-handler` (DaemonSet) |
| VM snapshot | `VirtualMachineSnapshot` CR |
| vMotion (live migration) | `VirtualMachineInstanceMigration` CR |
| OVA / OVF import | CDI `DataVolume` with HTTP/registry source |
| Distributed vSwitch (VDS) | OVN-Kubernetes + Multus + Linux Bridge CNI |
| NSX-T micro-segmentation | OCP NetworkPolicy + optional NMState |
| VMFS / vSAN datastore | OpenShift Data Foundation (ODF) / StorageClass |
| Content Library | `DataSource` + `DataImportCron` in openshift-virtualization-os-images |
| HA / DRS cluster | Kubernetes scheduler + pod disruption budgets |

The mapping isn't always 1:1, but having this vocabulary bridge makes searching documentation and interpreting error messages much faster — especially in the early weeks of a project.

---

### 3. Third-Party CNI / CSI Registries — Get the URLs Before You Start

If your architecture calls for a third-party CNI (e.g. Calico, Cilium, SR-IOV Network Operator) or a CSI driver (e.g. NetApp Trident, Pure Storage PSO, Dell CSI), **collect every image registry and endpoint those components need before you raise your first firewall change request**.

This is one of the most common causes of project delays in airgapped deployments:

```
Week 1  → OCP base cluster installs fine (Red Hat domains whitelisted)
Week 3  → OpenShift Virtualization installs fine (same domain set)
Week 5  → Calico operator fails — pulls from quay.io/tigera/...
          Firewall change request raised. Change window: 2 weeks.
Week 7  → Calico up, but Trident fails — pulls from docker.io/netapp/...
          Another change request. Another 2 weeks.
Week 9  → You're 4 weeks behind schedule.
```

Do a **single, comprehensive image audit at the start of the project**. For each third-party component, check:

1. **Operator bundle images** — what registry does OLM pull the operator itself from?
2. **Operand images** — what registry do the pods the operator deploys pull from?
3. **Init/sidecar images** — often overlooked; grep the operator's CSV for `image:` fields
4. **Helm chart images** — if the component ships via Helm, check `values.yaml` for every `repository:` key

A quick way to extract all image references from an operator bundle before installing:

```bash
# Pull the operator bundle and inspect all image references
oc mirror list operators \
  --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.17 \
  --package=sriov-network-operator \
  --channel=stable

# For a Helm chart, extract every unique registry
helm template <release> <chart> | grep 'image:' | sort -u
```

Common third-party registries that appear in OpenShift Virtualization adjacent deployments:

| Component | Registry |
|---|---|
| Calico / Tigera | `quay.io/tigera`, `docker.io/calico` |
| Cilium | `quay.io/cilium` |
| SR-IOV (upstream) | `ghcr.io/k8snetworkplumbingwg` |
| NetApp Trident | `docker.io/netapp` |
| Rook/Ceph | `quay.io/ceph`, `quay.io/cephcsi` |
| Longhorn | `docker.io/longhornio` |
| Cert-manager | `quay.io/jetstack` |
| MetalLB | `quay.io/metallb` |

Note that `docker.io` (Docker Hub) has rate limiting even before you consider firewall rules. If any of your components pull from Docker Hub, factor in either a paid Docker account, a mirror, or an explicit pull-through cache — and make sure that's in the firewall request too.

> **Bottom line:** Treat the registry/URL audit as a project deliverable on day one, not a task for when things start failing.

---

## References

- [Disconnected OpenShift Virtualization Made Easy — Red Hat Developer (August 2025)](https://developers.redhat.com/articles/2025/08/15/disconnected-openshift-virtualization-made-easy)
- [Simplify OpenShift Installation in Air-Gapped Environments — Red Hat Developer (October 2025)](https://developers.redhat.com/articles/2025/10/14/simplify-openshift-installation-air-gapped-environments)
- [Deploying Red Hat OpenShift Operators in a Disconnected Environment — Red Hat Blog](https://www.redhat.com/en/blog/deploying-red-hat-openshift-operators-disconnected-environment)
- [Red Hat OpenShift Disconnected Installations — Red Hat Blog](https://www.redhat.com/en/blog/red-hat-openshift-disconnected-installations)
- [Mirroring Images Using oc-mirror Plugin (OCP 4.16) — Red Hat Docs](https://docs.openshift.com/container-platform/4.16/installing/disconnected_install/installing-mirroring-disconnected.html)
- [OpenShift Virtualization Architecture (OCP 4.12) — Red Hat Docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.12/html/virtualization/virt-architecture)
- [HyperConverged Operator — KubeVirt.io](https://kubevirt.io/2019/Hyper-Converged-Operator.html)
- [Red Hat OpenShift Virtualization Service — IBM Cloud](https://www.ibm.com/products/openshift-virtualization)
- [Installing MAS on a Disconnected OpenShift Cluster — IBM Docs](https://www.ibm.com/docs/en/mas-cd?topic=setup-installing-disconnected-red-hat-openshift-cluster)
- [Using oc-mirror on IBM Z and LinuxONE — IBM Community](https://community.ibm.com/community/user/ibmz-and-linuxone/viewdocument/using-the-oc-mirror-plugin-for-disc)
- [Install Operators on Disconnected Clusters — Red Hat Marketplace / IBM](https://swc.saas.ibm.com/en-us/redhat-marketplace/documentation/deploy-products-to-a-disconnected-environment)
- [Hosted Control Planes on OpenShift Virtualization in Disconnected Environments — OKD 4.18 Docs](https://docs.okd.io/4.18/hosted_control_planes/hcp-disconnected/hcp-deploy-dc-virt.html)
