Here’s a review + a cleaned-up version you can paste back into your doc. I focused on: accuracy vs Microsoft guidance, clearer flow for “brownfield in-place migration,” consistent terminology, and fixing numbering/grammar.

## Key fixes to make

* **Terminology/formatting**

  * Use consistent casing/labels: **Network isolated (NI)**, **outboundType**, **bootstrap artifact source**, **Direct vs Cache**.
  * “Change Outbound type of none” → “Change **outboundType to `none`** (or `block` in preview).” NI supports outbound types `none` or `block`. ([Microsoft Learn][1])
* **Prereqs**

  * NI clusters are supported on **Kubernetes 1.30+** (per Microsoft). ([Microsoft Learn][1])
  * Only **NodeImage** and **None** upgrade channels are currently supported for NI clusters (and **Unmanaged** isn’t supported). ([Microsoft Learn][2])
* **Migration sequence (brownfield)**

  * Microsoft’s sequence for enabling NI on an existing cluster is:

    1. set **bootstrap-artifact-source = Cache**
    2. **reimage node pools**
    3. set **outbound-type = none**
       and keep outbound connectivity until the first reimage completes. ([Microsoft Learn][1])
  * Your doc says “3 rounds of node reimaging.” That might be **AT&T internal guidance**, but Microsoft docs clearly call out **at least** the reimage after switching to Cache (and again after other changes like ACR ID changes). I’d phrase this as “expect multiple reimages; minimum per Microsoft is X; AT&T plan uses Y.”
* **Storage note**

  * The custom StorageClass requirement for Azure Files/Blob CSI with `networkEndpointType: privateEndpoint` is correct per Microsoft. ([Microsoft Learn][2])
* **Validation**

  * Add both: (a) config validation (`az aks show`), and (b) functional validation (pods running, image pulls, add-ons, app endpoint tests).

---

## Revised draft (ready to paste)

### Table of contents

1. Introduction
2. Non-network-isolated AKS vs Network-isolated AKS (NI)
3. Prerequisites and required changes
4. Migration process (brownfield / in-place)
5. Pipeline run process
6. Validation
7. References

---

## 1. Introduction

Today our standard AKS clusters use **outboundType = `userDefinedRouting`** to force egress through **Azure Firewall**, where we apply FQDN-based restrictions. While this provides controlled egress, a **network isolated (NI) AKS cluster** strengthens a secure-by-default posture by removing outbound dependencies for **cluster bootstrapping** and allowing operators to explicitly enable only the outbound paths required for workloads and add-ons.

Network isolated clusters use a **private Azure Container Registry (ACR)** as the artifact source (**bootstrap artifact source = `Cache`**) so nodes can pull required components without needing outbound access to public registries during cluster bootstrap. ([Microsoft Learn][2])

This document summarizes the high-level changes and steps required to migrate an existing (brownfield) AKS cluster to NI.

---

## 2. Non-network-isolated AKS vs Network-isolated AKS

| Area                              | Non-NI AKS (typical today)                                                              | NI AKS                                                                                                                       |
| --------------------------------- | --------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **Outbound type**                 | `loadBalancer`, `managedNATGateway`, `userAssignedNATGateway`, **`userDefinedRouting`** | **`none`** (GA) or `block` (preview) ([Microsoft Learn][1])                                                                  |
| **Bootstrap artifact source**     | `Direct` (pull from Microsoft Artifact Registry / public endpoints)                     | `Cache` (pull via private ACR) ([Microsoft Learn][1])                                                                        |
| **Internet egress for bootstrap** | Usually required (commonly via Azure Firewall)                                          | Not required for bootstrapping; optional egress can be established later via Firewall/Proxy as needed ([Microsoft Learn][2]) |

> Note: The outbound type impacts **cluster egress traffic configuration**. For NI clusters, the supported outbound types are `none` and `block`. ([Microsoft Learn][2])

---

## 3. Prerequisites and required changes

### 3.1 Platform / AKS prerequisites

1. **AKS Kubernetes version must be 1.30 or higher**. ([Microsoft Learn][1])
2. **Upgrade channels**: only **`NodeImage`** and **`None`** are currently supported for NI clusters; **`Unmanaged`** is not supported. ([Microsoft Learn][2])
3. Ensure the team is aligned on whether outboundType will be:

   * **`none`** (AKS does not configure egress; customer sets explicit egress paths if needed) ([Microsoft Learn][2])
   * or `block` (preview; blocks all outbound) ([Microsoft Learn][2])

### 3.2 Registry / image sourcing changes

4. Ensure **all platform workloads and add-ons** pull images from approved internal registries (e.g., **JFrog / private ACR**) and not public registries.
5. Add ACR artifact cache rules as required to support pulling from Microsoft registries via ACR cache (example command in process section).

### 3.3 Storage changes (if applicable)

6. If using **Azure Files / Blob CSI**, create a custom StorageClass with `networkEndpointType: privateEndpoint` and plan for any PVC migration/recreation required. ([Microsoft Learn][2])

   * Action item: extend StorageClass “custom layer” templates to support SC parameters for dynamic PVs (where applicable).

### 3.4 Operational readiness

7. **PDB safety:** Identify PodDisruptionBudgets with **allowed disruptions = 0** that could block node drain during reimage. Backup and temporarily remove them before migration; restore after completion.
8. Compile a list of **application endpoints** to validate pre/post migration (health checks, downstream connectivity, private endpoints, etc.).
9. If environment requires retaining **Azure Firewall** (e.g., Mobility Transit / explicit egress needs), document the business justification and follow the internal exception process. (NI still uses outboundType `none`; firewall can remain for explicitly required egress paths.) ([Microsoft Learn][2])

> Internal guidance note (AT&T): If your migration plan requires multiple reimage passes to reduce risk, document the rationale and what changes each pass covers. Microsoft’s minimum requirement for enabling NI on an existing cluster includes reimaging after switching to `Cache`, and keeping outbound available until that first reimage completes. ([Microsoft Learn][1])

---

## 4. Migration process (brownfield / in-place)

### Step 1 — Create ACR cache rule(s) (example)

```bash
az acr cache create \
  -n aks-managed-mcr \
  -r ${REGISTRY_NAME} \
  -g ${RESOURCE_GROUP} \
  --source-repo "mcr.microsoft.com/*" \
  --target-repo "aks-managed-repository/*"
```

### Step 2 — Detect workloads still pulling from public registries

```bash
kubectl get pods,daemonsets,deployments,statefulsets --all-namespaces \
  --field-selector metadata.namespace!=kube-system,metadata.namespace!=kube-public \
  -o custom-columns="NAMESPACE:.metadata.namespace,KIND:.kind,NAME:.metadata.name,IMAGE:.spec.containers[*].image" \
| grep -v '\.azurecr\.io' \
| grep -v '\.it\.att\.com' \
| sort | uniq
```

### Step 3 — Handle PDBs that can block draining (pre-work)

* Backup PDB manifests (YAML)
* Temporarily remove PDBs with `allowedDisruptions = 0`
* Restore after migration completes

### Step 4 — Enable NI bootstrap artifact source (Cache)

Run (typically via pipeline):

```bash
az aks update \
  --resource-group ${RESOURCE_GROUP} \
  --name ${AKS_NAME} \
  --bootstrap-artifact-source Cache \
  --bootstrap-container-registry-resource-id ${REGISTRY_ID}
```

> Microsoft note: You must keep outbound connectivity in place until the first reimage finishes. ([Microsoft Learn][1])

### Step 5 — Reimage all existing node pools

Manual reimage:

```bash
az aks upgrade \
  --resource-group ${RESOURCE_GROUP} \
  --name ${AKS_NAME} \
  --node-image-only
```

Wait for completion:

```bash
NODEPOOLS=$(az aks nodepool list --resource-group "${RESOURCE_GROUP}" --cluster-name "${AKS_NAME}" --query "[].name" -o tsv)
for NODEPOOL in $NODEPOOLS; do
  echo "Waiting for node pool $NODEPOOL to finish upgrading..."
  az aks nodepool wait --resource-group "${RESOURCE_GROUP}" --cluster-name "${AKS_NAME}" --name "$NODEPOOL" --updated
  echo "Node pool $NODEPOOL upgrade succeeded."
done
```

### Step 6 — Update outbound type to none (final NI posture)

Run (typically via pipeline):

```bash
az aks update \
  --resource-group ${RESOURCE_GROUP} \
  --name ${AKS_NAME} \
  --outbound-type none
```

This yields NI posture where AKS does not set up outbound connections by default (explicit egress can be added later via firewall/proxy/private endpoints as needed). ([Microsoft Learn][2])

---

## 5. Pipeline run process

Document what the pipeline automates vs what is manual. Suggested structure:

* **Pre-check stage:** AKS version >= 1.30, registries validated, PDB scan, endpoint inventory complete
* **Stage A (enable Cache):** run `az aks update ... --bootstrap-artifact-source Cache ...`
* **Manual step:** nodepool reimage + wait
* **Stage B (set outbound none):** run `az aks update ... --outbound-type none`
* **Post checks:** `az aks show`, workload readiness, app endpoint validation

---

## 6. Validation

### 6.1 Configuration validation

```bash
az aks show --resource-group ${RESOURCE_GROUP} --name ${AKS_NAME}
```

Confirm:

* `networkProfile.outboundType` is `none` (or `block` if used)
* `bootstrapProfile.artifactSource` is `Cache`
* `bootstrapProfile.containerRegistryId` points to expected ACR

These are the indicators Microsoft highlights for NI enablement. ([Microsoft Learn][1])

### 6.2 Functional validation (recommended)

* Node pools healthy after reimage; cluster and system pods stable
* Confirm workloads can pull images from approved registries
* Validate each application endpoint from the precompiled list (pre vs post)
* Validate any add-ons requiring egress are using private endpoints or approved explicit egress paths (firewall/proxy) ([Microsoft Learn][2])

---

## 7. References

* Internal: AT&T SharePoint — NI AKS Brownfield In-place Migration
* Microsoft Learn:

  * Network isolated clusters (concepts) ([Microsoft Learn][2])
  * Create / update NI cluster (how-to) ([Microsoft Learn][1])

---

If you want, I can also tighten this into a 1–2 page “runbook style” version (with a pre-check checklist and a single linear command sequence), but the above is already consistent with Microsoft’s published flow and makes your AT&T-specific requirements explicit.

[1]: https://learn.microsoft.com/en-us/azure/aks/network-isolated "Create a network isolated AKS cluster - Azure Kubernetes Service | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/azure/aks/concepts-network-isolated "Network isolated AKS clusters - Azure Kubernetes Service | Microsoft Learn"
