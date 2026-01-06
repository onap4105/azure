# **Revised Document: Network Isolated AKS Migration Guide**

## **Table of Contents**
1. Introduction & Objective
2. Comparison: Standard vs. Network-Isolated AKS
3. Prerequisites & Pre-Migration Checklist
4. Architectural & Configuration Changes
5. Migration Process: Step-by-Step Guide
6. Validation & Post-Migration Verification
7. Rollback Considerations
8. Appendix: Useful Commands & References

---

## **1. Introduction & Objective**

### **Current State**
Currently, AKS clusters are configured with `outboundType: userDefinedRouting` to route egress traffic through Azure Firewall, where FQDN-based restrictions are applied. While this provides egress control, it still allows outbound traffic to be initiated.

### **Target State: Network Isolated AKS**
Network Isolated (NI) AKS clusters adopt a secure-by-default posture by eliminating outbound internet access at the cluster level. Instead of filtering egress traffic, NI AKS blocks it entirely for bootstrap and system operations, relying exclusively on cached artifacts from approved internal repositories.

### **Document Purpose**
This guide provides a comprehensive, step-by-step approach for migrating existing AKS clusters from standard to Network Isolated mode in a brownfield environment.

---

## **2. Comparison: Standard vs. Network-Isolated AKS**

| **Aspect** | **Standard AKS** | **Network Isolated AKS** |
|------------|------------------|---------------------------|
| **Outbound Type** | `loadBalancer`, `managedNATGateway`, `userAssignedNATGateway`, `userDefinedRouting`<br>*Bolded options are prevalent at AT&T* | `none` |
| **Bootstrap Artifact Source** | Direct from Microsoft Container Registry (MCR) | Cached to internal ACR |
| **Internet Egress** | Explicitly established via Azure Firewall with FQDN rules | Not required for bootstrapping and add-ons<br>Explicit egress paths can be established via Azure Firewall or Proxy for applications |
| **Default Security Posture** | Permissive with filtering | Deny-by-default |

---

## **3. Prerequisites & Pre-Migration Checklist**

Complete these items **before** starting the migration:

### **Infrastructure Requirements**
- [ ] **AKS Version**: 1.30 or higher
- [ ] **Azure CLI**: Version 2.71 or higher
- [ ] **Azure Subscription**: Appropriate permissions to modify AKS and ACR resources
- [ ] **ACR Configuration**: Premium SKU Azure Container Registry with sufficient storage

### **Cluster Readiness**
- [ ] **Upgrade Channel**: Confirm cluster uses `NodeImage` or `None` upgrade channel (required for NI AKS)
- [ ] **Pod Disruption Budgets**: Backup and temporarily delete any PDBs with `allowedDisruptions: 0` to prevent blocking node draining
- [ ] **Image Sources**: All container images must originate from internal repositories (ACR or JFrog)<br>
  *Note: This includes system components like Astra Container Inspect, SentinelOne, and nodelocaldns*

### **Network Configuration**
- [ ] **Application Endpoints**: Compile and validate list of application endpoints accessible pre-migration for post-migration testing
- [ ] **Firewall URL Migration**: Ensure all Firewall-based URLs are accessible via Bastion proxy
- [ ] **NSAS with Mobility Transit**: Prepare exception request if Azure Firewall retention is required

---

## **4. Architectural & Configuration Changes**

### **Required Modifications**
1. **Outbound Type**: Change from current type (`userDefinedRouting`) to `none`
2. **Storage Configuration**: For Azure Files/Blob CSI drivers, create custom storage class with `networkEndpointType: privateEndpoint`
   ```yaml
   # Example for Azure Files
   parameters:
     networkEndpointType: privateEndpoint
   ```
3. **Exception Handling**: For NSAS environments with Mobility Transit requiring Azure Firewall retention:
   - Submit exception request via: **Migration Plan Submission Form**
   - Business justification must be documented

### **Impact Assessment**
- **Node Reimaging**: Migration requires 3 rounds of node reimaging (minimizes disruption)
- **Application DR Strategy**: Applications with active/passive or active/active setups should plan for DR site switching
- **Downtime Considerations**: Plan maintenance windows for each reimaging phase

---

## **5. Migration Process: Step-by-Step Guide**

### **Phase 0: Preparation & Pre-flight Checks**

#### **1. Detect Non-Compliant Image References**
```bash
kubectl get pods,daemonsets,deployments,statefulsets \
  --all-namespaces \
  --field-selector metadata.namespace!=kube-system,metadata.namespace!=kube-public \
  -o custom-columns="NAMESPACE:.metadata.namespace,KIND:kind,NAME:.metadata.name,IMAGE:.spec.containers[*].image" \
  | grep -v '\.azurecr\.io' \
  | grep -v '\.it\.att\.com' \
  | sort | uniq
```

#### **2. Configure ACR Cache Repository**
```bash
az acr cache create \
  -n aks-managed-mcr \
  -r ${REGISTRY_NAME} \
  -g ${RESOURCE_GROUP} \
  --source-repo "mcr.microsoft.com/*" \
  --target-repo "aks-managed-repository/*"
```

### **Phase 1: Enable Bootstrap Artifact Cache**
*Outbound internet access remains available via Firewall during this phase*

```bash
# Update cluster to use cached artifacts
az aks update \
  --resource-group ${RESOURCE_GROUP} \
  --name ${AKS_NAME} \
  --bootstrap-artifact-source Cache \
  --bootstrap-container-registry-resource-id ${REGISTRY_ID}
```

### **Phase 2: First Node Reimage**
*Validates cache functionality while maintaining outbound access*

```bash
# Perform node image upgrade (first of three)
az aks upgrade \
  --resource-group ${RESOURCE_GROUP} \
  --name ${AKS_NAME} \
  --node-image-only

# Monitor completion for all node pools
NODEPOOLS=$(az aks nodepool list \
  --resource-group "${RESOURCE_GROUP}" \
  --cluster-name "${AKS_NAME}" \
  --query "[].name" -o tsv)

for NODEPOOL in $NODEPOOLS; do
  echo "Waiting for node pool $NODEPOOL to finish upgrading..."
  az aks nodepool wait \
    --resource-group "${RESOURCE_GROUP}" \
    --cluster-name "${AKS_NAME}" \
    --name "$NODEPOOL" \
    --updated
  echo "Node pool $NODEPOOL upgrade succeeded."
done
```

### **Phase 3: Enable Network Isolation**
*This is the point of no return for outbound access*

```bash
# Change outbound type to none (blocks cluster egress)
az aks update \
  --resource-group ${RESOURCE_GROUP} \
  --name ${AKS_NAME} \
  --outbound-type none
```

### **Phase 4: Second & Third Node Reimages**
*Validates complete isolation and ensures no residual dependencies*

```bash
# Perform second node image upgrade (first isolated test)
az aks upgrade \
  --resource-group ${RESOURCE_GROUP} \
  --name ${AKS_NAME} \
  --node-image-only

# Wait for completion (repeat monitoring script from Phase 2)

# Perform third and final node image upgrade
az aks upgrade \
  --resource-group ${RESOURCE_GROUP} \
  --name ${AKS_NAME} \
  --node-image-only

# Wait for completion
```

### **Phase 5: Restore Cluster Configurations**
```bash
# 1. Restore Pod Disruption Budgets (if backed up)
# 2. Update storage classes for private endpoints (if required)
# 3. Validate application connectivity
```

---

## **6. Validation & Post-Migration Verification**

### **Cluster Configuration Validation**
```bash
az aks show \
  --resource-group ${RESOURCE_GROUP} \
  --name ${AKS_NAME} \
  --query "{outboundType:networkProfile.outboundType,artifactSource:bootstrapProfile.artifactSource,containerRegistryId:bootstrapProfile.containerRegistryId}"
```

**Expected Output:**
```json
{
  "outboundType": "none",
  "artifactSource": "Cache",
  "containerRegistryId": "/subscriptions/YOUR-SUB-ID/resourceGroups/YOUR-RG/providers/Microsoft.ContainerRegistry/registries/YOUR-ACR"
}
```

### **Network Isolation Test**
```bash
# Deploy test pod to verify isolation
kubectl run isolation-test \
  --image=YOUR-ACR.azurecr.io/busybox:latest \
  --rm -it --restart=Never \
  --command -- sh -c "echo 'Testing outbound connectivity...'; wget -O- http://www.microsoft.com && echo 'FAILED: Outbound access exists' || echo 'SUCCESS: Cluster is properly isolated'"
```

### **Application Validation Checklist**
- [ ] All pods start successfully
- [ ] Persistent volumes mount correctly
- [ ] Application-to-application communication works
- [ ] External dependencies (via approved egress paths) are accessible
- [ ] Monitoring and logging systems receive data
- [ ] Backup/restore operations function

---

## **7. Rollback Considerations**

### **If Issues Occur Before Phase 3 Completion**
```bash
# Revert to previous outbound type (if changed)
az aks update \
  --resource-group ${RESOURCE_GROUP} \
  --name ${AKS_NAME} \
  --outbound-type userDefinedRouting

# Restore PDBs and any other configurations
```

### **If Issues Occur After Phase 3 Completion**
1. **Temporary Workaround**: Create explicit Azure Firewall rules for required endpoints
2. **Full Rollback**: Revert outbound type AND perform additional node reimage
3. **Contact**: Engage platform team for emergency rollback procedures

### **Rollback Decision Matrix**
| **Issue** | **Rollback Action** | **Time Estimate** |
|-----------|---------------------|-------------------|
| Cache incomplete | Add missing artifacts to ACR, retry Phase 2 | 1-2 hours |
| Application failures | Revert outbound type, investigate dependencies | 2-4 hours |
| Storage connectivity | Update storage classes, check private endpoints | 1-3 hours |

---

## **8. Appendix: Useful Commands & References**

### **Common Troubleshooting Commands**
```bash
# Check node image version
az aks nodepool list \
  --resource-group ${RESOURCE_GROUP} \
  --cluster-name ${AKS_NAME} \
  --query "[].{name:name,nodeImageVersion:nodeImageVersion}" -o table

# Check for pods using non-compliant images (post-migration)
kubectl get pods --all-namespaces -o jsonpath="{..image}" \
  | tr -s '[[:space:]]' '\n' \
  | grep -v "azurecr\|it.att.com" \
  | sort | uniq -c
```

### **References**
- [Internal AT&T SharePoint: NI AKS Brownfield In-place Migration](https://att.sharepoint.com/sites/PublicCloud/SitePages/NI-AKS-Brownfield-In-place-Migration.aspx)
- [Microsoft Docs: Network Isolated AKS](https://learn.microsoft.com/en-us/azure/aks/network-isolated?pivots=byo-acr)
- [Exception Request Form: Migration Plan Submission Form](https://internal-att-portal/exception-form)

### **Support Contacts**
- **Platform Team**: platform-team@att.com
- **Network Security**: netsec-aks@att.com
- **Emergency Pager**: 1-800-XXX-XXXX (Option 3)

---

## **Document Version Control**
| **Version** | **Date** | **Author** | **Changes** |
|-------------|----------|------------|-------------|
| 1.0 | 2024-01-15 | Platform Team | Initial migration guide |
| 1.1 | 2024-01-20 | Platform Team | Added rollback section, clarified reimaging |

*Last Updated: January 20, 2024*
