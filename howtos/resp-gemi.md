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

Here is a comprehensive Bash validation script. It automates the checks for **CLI versions, AKS versions, Upgrade Channels, Pod Disruption Budgets (PDBs), and Public Image References**.

Save this script as `validate_ni_migration.sh`.

### **Script: `validate_ni_migration.sh**`

```bash
#!/bin/bash

# ==============================================================================
# Network Isolated AKS Migration - Pre-Flight Validation Script
# ==============================================================================
# This script validates the readiness of an AKS cluster for Network Isolation migration.
# It checks:
# 1. Azure CLI Version (>= 2.71)
# 2. AKS Cluster Version (>= 1.30)
# 3. Upgrade Channel (NodeImage or None)
# 4. Pod Disruption Budgets (PDBs) blocking eviction
# 5. Non-compliant Image References (Public Internet pulls)
# ==============================================================================

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Check for required tools
command -v az >/dev/null 2>&1 || { echo -e "${RED}Error: 'az' CLI is not installed.${NC}"; exit 1; }
command -v kubectl >/dev/null 2>&1 || { echo -e "${RED}Error: 'kubectl' is not installed.${NC}"; exit 1; }
command -v jq >/dev/null 2>&1 || { echo -e "${RED}Error: 'jq' is not installed. Please install it to run this script.${NC}"; exit 1; }

# Usage check
if [ "$#" -ne 2 ]; then
    echo -e "${YELLOW}Usage: $0 <RESOURCE_GROUP> <CLUSTER_NAME>${NC}"
    echo "Example: $0 my-rg my-aks-cluster"
    exit 1
fi

RESOURCE_GROUP=$1
CLUSTER_NAME=$2
FAILURES=0
WARNINGS=0

echo -e "\n${BLUE}=== Starting Pre-Flight Checks for Cluster: ${CLUSTER_NAME} ===${NC}"

# ------------------------------------------------------------------------------
# 1. Validate Azure CLI Version
# ------------------------------------------------------------------------------
echo -e "\n${BLUE}[1/5] Checking Azure CLI Version...${NC}"
REQUIRED_CLI="2.71"
CURRENT_CLI=$(az version --output json | jq -r '."azure-cli"')

# Version comparison logic
if [ "$(printf '%s\n' "$REQUIRED_CLI" "$CURRENT_CLI" | sort -V | head -n1)" = "$REQUIRED_CLI" ]; then
    echo -e "${GREEN}PASS: Azure CLI version is $CURRENT_CLI (>= $REQUIRED_CLI)${NC}"
else
    echo -e "${RED}FAIL: Azure CLI version is $CURRENT_CLI. Must be >= $REQUIRED_CLI${NC}"
    ((FAILURES++))
fi

# ------------------------------------------------------------------------------
# 2. Validate AKS Cluster Configuration
# ------------------------------------------------------------------------------
echo -e "\n${BLUE}[2/5] Checking AKS Cluster Configuration...${NC}"

# Fetch Cluster Details
CLUSTER_JSON=$(az aks show --resource-group "$RESOURCE_GROUP" --name "$CLUSTER_NAME" --output json 2>/dev/null)

if [ -z "$CLUSTER_JSON" ]; then
    echo -e "${RED}Error: Could not find cluster '$CLUSTER_NAME' in resource group '$RESOURCE_GROUP'.${NC}"
    exit 1
fi

# Check AKS Version
REQUIRED_AKS="1.30"
CURRENT_AKS=$(echo "$CLUSTER_JSON" | jq -r '.kubernetesVersion')

if [ "$(printf '%s\n' "$REQUIRED_AKS" "$CURRENT_AKS" | sort -V | head -n1)" = "$REQUIRED_AKS" ]; then
    echo -e "${GREEN}PASS: AKS Version is $CURRENT_AKS (>= $REQUIRED_AKS)${NC}"
else
    echo -e "${RED}FAIL: AKS Version is $CURRENT_AKS. Must be >= $REQUIRED_AKS${NC}"
    ((FAILURES++))
fi

# Check Upgrade Channel
UPGRADE_CHANNEL=$(echo "$CLUSTER_JSON" | jq -r '.upgradeChannel')
if [[ "$UPGRADE_CHANNEL" == "node-image" || "$UPGRADE_CHANNEL" == "none" || "$UPGRADE_CHANNEL" == "NodeImage" || "$UPGRADE_CHANNEL" == "None" ]]; then
    echo -e "${GREEN}PASS: Upgrade Channel is '$UPGRADE_CHANNEL'${NC}"
else
    echo -e "${RED}FAIL: Upgrade Channel is '$UPGRADE_CHANNEL'. Must be 'NodeImage' or 'None'.${NC}"
    ((FAILURES++))
fi

# ------------------------------------------------------------------------------
# 3. Validate Pod Disruption Budgets (PDBs)
# ------------------------------------------------------------------------------
echo -e "\n${BLUE}[3/5] Checking Pod Disruption Budgets (PDBs)...${NC}"
# Find PDBs with allowedDisruptions = 0
BLOCKING_PDBS=$(kubectl get pdb -A -o json | jq -r '.items[] | select(.status.disruptionsAllowed==0) | "\(.metadata.namespace)/\(.metadata.name)"')

if [ -z "$BLOCKING_PDBS" ]; then
    echo -e "${GREEN}PASS: No PDBs found blocking node drainage (Allowed Disruptions > 0).${NC}"
else
    echo -e "${YELLOW}WARNING: The following PDBs have 0 allowed disruptions and will block node reimaging:${NC}"
    echo "$BLOCKING_PDBS"
    echo -e "${YELLOW}Action: Backup and delete these PDBs before migration.${NC}"
    ((WARNINGS++))
fi

# ------------------------------------------------------------------------------
# 4. Check for Public Image References
# ------------------------------------------------------------------------------
echo -e "\n${BLUE}[4/5] Scanning for Public Image References...${NC}"
echo "Searching for images NOT from .azurecr.io or .it.att.com..."

# Fetch all images, filter out valid ones, and list the offenders
NON_COMPLIANT_IMAGES=$(kubectl get pods,daemonsets,deployments,statefulsets --all-namespaces \
  --field-selector metadata.namespace!=kube-system,metadata.namespace!=kube-public \
  -o custom-columns="NAMESPACE:.metadata.namespace,KIND:kind,NAME:.metadata.name,IMAGE:.spec.containers[*].image" \
  | grep -v 'IMAGE' \
  | grep -v '\.azurecr\.io' \
  | grep -v '\.it\.att\.com' \
  | sort | uniq)

if [ -z "$NON_COMPLIANT_IMAGES" ]; then
    echo -e "${GREEN}PASS: All detected images are pulling from private registries.${NC}"
else
    echo -e "${RED}FAIL: Found workloads using public/unauthorized images:${NC}"
    echo "---------------------------------------------------"
    echo "$NON_COMPLIANT_IMAGES"
    echo "---------------------------------------------------"
    echo -e "${RED}Action: Retag these images to your private ACR or JFrog before proceeding.${NC}"
    ((FAILURES++))
fi

# ------------------------------------------------------------------------------
# 5. Check Storage Classes (Informational)
# ------------------------------------------------------------------------------
echo -e "\n${BLUE}[5/5] Checking Storage Classes (Advisory)...${NC}"
CSI_DRIVERS=$(kubectl get sc -o json | jq -r '.items[] | select(.provisioner | contains("csi.azure")) | .metadata.name')

if [ ! -z "$CSI_DRIVERS" ]; then
    echo -e "${YELLOW}WARNING: Azure CSI Storage Classes detected: $CSI_DRIVERS${NC}"
    echo -e "${YELLOW}Reminder: Ensure you have created custom StorageClasses with 'networkEndpointType: privateEndpoint'.${NC}"
    ((WARNINGS++))
else
    echo -e "${GREEN}PASS: No obvious Azure CSI storage classes found (or check skipped).${NC}"
fi

# ------------------------------------------------------------------------------
# Summary
# ------------------------------------------------------------------------------
echo -e "\n${BLUE}=== Validation Summary ===${NC}"
if [ $FAILURES -eq 0 ]; then
    if [ $WARNINGS -eq 0 ]; then
        echo -e "${GREEN}SUCCESS: Cluster is READY for Network Isolated migration.${NC}"
    else
        echo -e "${YELLOW}READY WITH WARNINGS: Fix the warnings above (PDBs/Storage) before starting pipeline.${NC}"
    fi
else
    echo -e "${RED}FAILED: Please fix the $FAILURES errors listed above before attempting migration.${NC}"
    exit 1
fi

```

### **How to Use This Script**

1. **Save the file:**
Create a file named `validate_ni_migration.sh` and paste the code above into it.
2. **Make it executable:**
```bash
chmod +x validate_ni_migration.sh

```


3. **Run the script:**
You need to provide your **Resource Group** and **AKS Cluster Name**.
```bash
./validate_ni_migration.sh <RESOURCE_GROUP> <AKS_CLUSTER_NAME>

```


*Example:*
```bash
./validate_ni_migration.sh my-rg-01 my-aks-cluster-01

```



### **What This Script Does**

* **Version Check:** Ensures your `az` CLI is new enough to support the cache commands and your AKS cluster is v1.30+.
* **Config Check:** Verifies the upgrade channel is correct (`NodeImage` is critical for the manual reimage steps).
* **PDB Safety:** Scans for "Immutable" Pods (where `allowedDisruptions: 0`). These will cause the node drain process to hang indefinitely if not removed.
* **Image Security:** It runs the specific `kubectl` command from your document to find any image that isn't hosted on your private ACR (`*.azurecr.io`) or internal IT repo (`*.it.att.com`).
* **Storage Warning:** It scans for Azure CSI storage classes to remind you about the `privateEndpoint` requirement mentioned in your prerequisites.

Would you like me to create a **validation script** (in Bash or Python) that automates the "Pre-Flight Checks" (Step 2 of the process) so users can run a single command to see if their cluster is ready for migration?
