This is a solid draft for a technical migration document. It correctly identifies the core concept (moving from `userDefinedRouting` to `none` outbound type) and the operational steps required.

However, the current format is a bit disjointed. It mixes prerequisites, strategic decisions (like DR sites), and tactical commands into a single list. The "Process" section also jumps between pipeline concepts and manual CLI commands without clear separation.

Below is a **reviewed and polished version** of your document. I have restructured it to be cleaner, clearer, and more actionable for an engineering team, while fixing some minor grammatical issues.

---

### **Review Summary**

* **Structure:** Separated "Prerequisites" from the "Migration Process" to prevent confusion between what must be done *before* starting versus what is done *during* the migration.
* **Clarity:** The distinction between "Non-Network Isolated" and "Network Isolated" is now a clear table.
* **Commands:** Formatted all CLI and `kubectl` commands into code blocks for easy copy-pasting.
* **Gap Identified:** Step 4 mentions "3 rounds of node reimaging," but the process steps only explicitly show one manual reimage and one implicit pipeline run. I have clarified this in the process flow.

---

# Polished Document: Network Isolated AKS Migration Guide

## Table of Contents

1. **Introduction**
2. **Comparison: Non-Network Isolated vs. Network Isolated**
3. **Prerequisites & Pre-Migration Checklist**
4. **Migration Process**
* ACR Configuration
* Pre-Flight Checks
* Migration Execution (Pipeline & CLI)


5. **Validation**
6. **References**

---

## 1. Introduction

Currently, we configure the cluster outbound type to `userDefinedRouting` to force outbound traffic through the Azure Firewall, applying FQDN restrictions on egress. While Azure Firewall defines egress restrictions, **Network Isolated (NI) clusters** advance our "secure-by-default" posture by eliminating or blocking outbound requests altogether.

This document outlines the high-level changes and technical steps required to migrate existing clusters to a Network Isolated AKS architecture.

---

## 2. Comparison: Non-Network Isolated vs. Network Isolated

| Feature | Non-Network Isolated AKS | Network Isolated AKS |
| --- | --- | --- |
| **Outbound Types** | `loadBalancer`, `managedNATGateway`, `userAssignedNATGateway`, `userDefinedRouting` | `none` |
| **Bootstrap Artifact Source** | Direct (Public Internet) | Cache (Private Registry) |
| **Internet Egress** | Explicitly established with Azure Firewall | Not required for bootstrapping or add-ons. Explicit egress paths can still be established via Proxy or Azure Firewall if needed. |

> **Note:** The outbound type impacts only the egress traffic of the cluster.

---

## 3. Prerequisites & Pre-Migration Checklist

Before initiating the migration pipeline, ensure the following requirements are met:

### **Cluster & Tools**

* **AKS Version:** Must be **1.30 or above**.
* **CLI Version:** `az` CLI version **2.71+**.
* **Upgrade Channels:** Only `NodeImage` and `None` upgrade channels are supported for NI clusters.

### **Storage & Networking**

* **CSI Driver Update:** If using Azure CSI for Files/Blob, you must migrate to a custom StorageClass with `networkEndpointType: privateEndpoint`.
* *Action:* Update StorageClass custom Layer types to include parameters for dynamic PersistentVolumes.
* *Action:* Recreate PVCs with the new StorageClass.


* **Proxy Configuration:** Ensure all current Firewall-based URLs are routed through the **Bastion Proxy** instead of the Azure Firewall.
* **Endpoint Validation:** Compile a list of application endpoints to validate connectivity both pre- and post-implementation.

### **Operational Readiness**

* **Artifact Registry:** Ensure all platform deployments (including Astra, SentinelOne, NodeLocalDNS, etc.) are updated to use **JFrog** or the private registry.
* **Disaster Recovery:** Applications with active/passive or canary setups should switch traffic to the DR site during migration.
* **PDB Management:** PodDisruptionBudgets (PDBs) with `allowedDisruptions: 0` must be backed up and deleted to prevent blocking node drainage. Restore them after migration.
* **NSAS Exception:** For NSAS environments requiring Mobility Transit, you must retain Azure Firewall. Convert to NI (outbound type `none`) and submit an exception request via the **Migration Plan Submission Form**.

---

## 4. Migration Process

### **Phase 1: ACR Configuration**

Add the new cache repo to your Azure Container Registry (ACR):

```bash
az acr cache create -n aks-managed-mcr \
  -r ${REGISTRY_NAME} \
  -g ${RESOURCE_GROUP} \
  --source-repo "mcr.microsoft.com/*" \
  --target-repo "aks-managed-repository/*"

```

### **Phase 2: Detect Public Image References**

Verify that no workloads are pulling images from the public internet. Run the following to detect non-compliant images:

```bash
kubectl get pods,daemonsets,deployments,statefulsets --all-namespaces \
  --field-selector metadata.namespace!=kube-system,metadata.namespace!=kube-public \
  -o custom-columns="NAMESPACE:.metadata.namespace,KIND:kind,NAME:.metadata.name,IMAGE:.spec.containers[*].image" \
  | grep -v '\.azurecr\.io' \
  | grep -v '\.it\.att\.com' \
  | sort | uniq

```

### **Phase 3: Migration Execution**

**Step 1: Backup PDBs**
If your cluster has PDBs with `0` allowed disruptions, back them up and delete them now.

**Step 2: Enable Artifact Caching (Pipeline Run)**
Update the cluster to use the cached artifact source.

```bash
az aks update --resource-group ${RESOURCE_GROUP} --name ${AKS_NAME} \
  --bootstrap-artifact-source Cache \
  --bootstrap-container-registry-resource-id ${REGISTRY_ID}

```

**Step 3: First Node Reimage (Manual)**
Manually reimage existing nodepools to apply the bootstrap configuration.

```bash
az aks upgrade --resource-group ${RESOURCE_GROUP} --name ${AKS_NAME} --node-image-only

```

**Step 4: Verify Reimage Completion**
*Crucial: You must ensure outbound connectivity exists until this reimage completes.*

```bash
NODEPOOLS=$(az aks nodepool list --resource-group "${RESOURCE_GROUP}" --cluster-name "${AKS_NAME}" --query "[].name" -o tsv)

for NODEPOOL in $NODEPOOLS; do
  echo "Waiting for node pool $NODEPOOL to finish upgrading..."
  az aks nodepool wait --resource-group "${RESOURCE_GROUP}" --cluster-name "${AKS_NAME}" --name "$NODEPOOL" --updated
  echo "Node pool $NODEPOOL upgrade succeeded."
done

```

**Step 5: Switch Outbound Type (Pipeline Run)**
Once the reimage is confirmed, update the outbound type to `none`.

```bash
az aks update --resource-group ${RESOURCE_GROUP} --name ${AKS_NAME} --outbound-type none

```

*(Note: As per requirements, this migration may require up to 3 rounds of node reimaging in total to fully stabilize. Ensure Step 3 is repeated if necessary based on specific environment constraints.)*

---

## 5. Validation

To validate that the network isolated cluster feature is enabled, run:

```bash
az aks show --resource-group ${RESOURCE_GROUP} --name ${AKS_NAME}

```

**Expected Output:**
Review the JSON output for `outboundType: "none"` and `artifactSource: "Cache"`.

```json
{
  "properties": {
    "networkProfile": {
      "outboundType": "none"
    },
    "bootstrapProfile": {
      "artifactSource": "Cache",
      "containerRegistryId": "/subscriptions/.../registries/my-registry-name"
    }
  }
}

```

---

## 6. References

* [Internal SharePoint: NI AKS Brownfield In-place Migration](https://att.sharepoint.com/sites/PublicCloud/SitePages/NI-AKS-Brownfield-In-place-Migration.aspx)
* [Microsoft Docs: Network Isolated AKS (BYO ACR)](https://learn.microsoft.com/en-us/azure/aks/network-isolated?pivots=byo-acr)

---

### **Next Step for You**

Would you like me to create a **validation script** (in Bash or Python) that automates the "Pre-Flight Checks" (Step 2 of the process) so users can run a single command to see if their cluster is ready for migration?
