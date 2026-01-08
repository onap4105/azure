In **network isolated AKS**, most of the “magic” is implemented by **node-level bootstrap + kubelet configuration** (via the node provisioning scripts / CSE). Changing cluster settings in Azure doesn’t fully “flip” existing nodes in-place — the new behavior becomes reliable only after nodes are **reimaged** so they rerun bootstrap with the new settings. That’s why teams often treat it as a **3-round reimage** to make the cutover deterministic.

### Round 1 — Reimage right after switching bootstrap artifacts to **Cache**

When you update the cluster to use **cached bootstrap artifacts**, AKS needs nodes to rebuild with kubelet/bootstrap pointing at the cached artifact source. Microsoft explicitly calls out that you must reimage immediately after switching `bootstrap-artifact-source` to `Cache`, otherwise the feature won’t take effect. ([Microsoft Learn][1])

**Why it’s crucial:** if you don’t reimage here, you end up with a “mixed reality” cluster: control-plane/cluster settings say “Cache”, but existing nodes still behave like “Direct”, and later steps will break image pulls and add-on bootstrap.

### Round 2 — Reimage after enabling/updating the network isolation wiring (ACR/private endpoints/DNS)

Network isolated clusters depend on **ACR cache rules + private connectivity** (AKS-managed ACR or BYO ACR). If you enabled the feature (or changed the private ACR ID), nodes must reimage so kubelet/CSE picks up the new registry/cache configuration; Microsoft’s troubleshooting guidance says image-pull failures after converting or updating ACR are *expected* until you reimage the nodes to update kubelet configuration. ([Microsoft Learn][2])

**Why it’s crucial:** this is where you eliminate “old kubelet config” problems that show up as intermittent `ImagePullBackOff` even though the cluster looks correctly configured.

### Round 3 — Reimage after switching outbound to **none** (or fully blocking egress)

Microsoft’s conversion flow explicitly says: keep outbound working until the first reimage completes, *then* set `outbound-type none`. ([Microsoft Learn][1])
In practice, many teams do one more reimage after the outbound cutover to ensure **every node** comes up clean in the *final* “no default egress” posture (routes/iptables/add-on restarts/cert pulls/etc. all happen under the new reality).

**Why it’s crucial:** if you change outbound too early (or without a final clean node cycle), some nodes/pods still rely on “old internet assumptions” and you get failures that are hard to debug because only part of the fleet is truly isolated.

---

**One-line summary:** the “3 rounds” are about preventing a partially-updated fleet — each round forces nodes to re-run bootstrap under the next isolation milestone so **image pulls, kubelet settings, and add-ons converge** before you remove egress completely.

[1]: https://learn.microsoft.com/en-us/azure/aks/network-isolated "Create a network isolated AKS cluster - Azure Kubernetes Service | Microsoft Learn"
[2]: https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/extensions/troubleshoot-network-isolated-cluster "Troubleshoot Network Isolated AKS Clusters - Azure | Microsoft Learn"
cornerstone of incident response in immutable, cloud-based infrastructure.
