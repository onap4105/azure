# AKS Egress Dependency Inventory Template

Purpose: Inventory every outbound (and key inbound) dependency before removing Azure Firewall and adopting the Network Isolated AKS pattern. Use this as a living document. Each row represents a logical dependency (service + domain + purpose). 

## 1. Primary Table (Minimal Columns)

|Category|Service / Function|FQDN / Domain|Protocol|Port(s)|Namespace|Workload|Access Mode|Private Link Available|Current Path|Future Path|Criticality|Usage Pattern|Plan|Phase|Status|Risk If Blocked|Mitigation|Owner Tech|Verified|
|--------|------------------|-------------|--------|-------|---------|--------|-----------|----------------------|------------|-----------|-----------|-------------|----|-----|------|---------------|----------|-----------|---------|
|Azure Core|AKS Control Plane|management.azure.com|HTTPS|443|N/A|Cluster Ops|Public|Yes (Service Tags)|Firewall|Service Tags Allow|Blocker|Continuous|Privatize|1|Validated|Cluster mgmt fails|Allow service tag|Platform Team|Y|
|Registry|ACR|myacr.azurecr.io|HTTPS|443|all|image pulls|PrivateLink|Yes|Firewall|PrivateLink|Blocker|Boot/Burst|Privatize|2|In Progress|Image pulls fail|Create PE + DNS|Platform|N|
|Secrets|Key Vault|mykv.vault.azure.net|HTTPS|443|prod|apps|PrivateLink|Yes|Firewall|PrivateLink|High|Periodic|Privatize|2|Planned|Secret fetch fail|Enable PE|Security|N|
|OS Updates|Ubuntu Archive|archive.ubuntu.com|HTTPS|80/443|nodepool|node|Public|No|Firewall|Mirror/Proxy|Medium|Boot/Periodic|Mirror|1|Planned|Patch lag|Apt proxy|Infra|N|
|Packages|PyPI|pypi.org|HTTPS|443|ml|trainer|Public|No|Firewall|Proxy|Medium|Build|Proxy|2|Planned|Build failures|Internal cache|DevEx|N|
|Monitoring|Log Analytics Ingest|*.ods.opinsights.azure.com|HTTPS|443|kube-system|ama-agent|PrivateLink|Yes (workspace)|Firewall|PrivateLink|High|Continuous|Privatize|2|In Progress|Logging loss|Enable PE|Observability|N|
|Certs|LetsEncrypt ACME|acme-v02.api.letsencrypt.org|HTTPS|443|ingress|cert-manager|Public|No|Firewall|EgressGW|High|Periodic|Keep Public via GW|3|Planned|Cert expiry|ACME DNS-01 alt|Platform|N|
|CRL/OCSP|DigiCert OCSP|ocsp.digicert.com|HTTP|80|all|various|Public|No|Firewall|EgressGW|Medium|Periodic|Keep Public|3|Planned|Revocation checks fail|Cache / allowlist|Security|N|
|GitHub|GitHub API|api.github.com|HTTPS|443|ci|runners|Public|No|Firewall|Proxy|Medium|Build|Proxy|2|Planned|CI fail|Enterprise proxy|DevOps|N|

Phase legend: 0-Discovery, 1-Hardening, 2-Egress Shrink, 3-Cutover, 4-Optimize

## 1a. Full Extended Table (All Columns)
Complete superset of columns for detailed tracking. Use this when you need the full operational view (may wrap horizontally in some viewers).

|Category|Service / Function|FQDN / Domain|Exact Host|IP|Protocol|Port(s)|Direction|AKS Layer|Namespace|Workload|Container Image|Image Registry|Access Mode|Private Link Available|Current Path|Future Path|Requires DNS|DNS Zone|Auth Method|Data Sensitivity|Criticality|Usage Pattern|Avg Frequency|Last Seen|Volume|Fallback / Cache|Can Mirror/Proxy|Plan|Migration Phase|Status|Risk If Blocked|Mitigation|Owner (Business)|Owner (Technical)|Approval Ticket / Ref|Policy Rule ID|NetPol / Cilium Policy Ref|Logging Source|Verification Method|Verified|Decommission Candidate|Notes|
|--------|------------------|-------------|----------|--|--------|-------|---------|---------|---------|--------|---------------|--------------|-----------|----------------------|------------|-----------|------------|--------|-----------|---------------|-----------|-------------|-------------|---------|------|---------------|---------------|----|---------------|------|---------------|----------------|------------------|------------------|----------------------|--------------|-----------------------|----------|----------------------|-----|----------------------|-----|
|Registry|ACR|myacr.azurecr.io|myacr.azurecr.io|10.10.5.4|HTTPS|443|Egress|Platform|all|image pulls|n/a (node kubelet)|myacr.azurecr.io|PrivateLink|Yes|Firewall|PrivateLink|Yes|Private|MSI|Low|Blocker|Boot/Burst|per pod start|2025-09-04T12:00Z|~500 pulls/day|N|N|Privatize|2|In Progress|Image pulls fail|Create PE + private DNS|Platform|Platform|NET-1234|PL-RULE-01|netpol-acr-egress|Flow/DNS|Synthetic pull|N|N|Add PE before firewall removal|
|Secrets|Key Vault|mykv.vault.azure.net|mykv.vault.azure.net|10.10.6.8|HTTPS|443|Egress|Workload|prod|app-deploy|app:1.2.3|myacr.azurecr.io|PrivateLink|Yes|Firewall|PrivateLink|Yes|Private|MSI|High|High|Periodic|hourly|2025-09-04T11:55Z|~72 req/day|N|N|Privatize|2|Planned|Secret fetch fail|Enable PE + test pod|Security|Security|SEC-567|PL-RULE-02|netpol-kv-egress|App|Synthetic curl|N|N|--|
|Monitoring|Log Analytics Ingest|*.ods.opinsights.azure.com|region.ods.opinsights.azure.com|10.10.7.10|HTTPS|443|Egress|Add-on|kube-system|ama-agent|ama:3.2.0|mcr.microsoft.com|PrivateLink|Yes (workspace)|Firewall|PrivateLink|Yes|Private|Key|Moderate|High|Continuous|per 15s|2025-09-04T12:00Z|~50 KB/s|N|N|Privatize|2|In Progress|Logging loss|Create PE + DNS zone|Observability|Observability|OBS-45|PL-RULE-03|netpol-logs-egress|Flow|Canary ingest|N|N|--|
|Packages|PyPI|pypi.org|files.pythonhosted.org|Public IP|HTTPS|443|Egress|Workload|ml|trainer|trainer:4.5.1|pypi.org|Proxy|No|Firewall|Proxy|Yes|Public|None|Low|Medium|Build|per build|2025-09-03T20:10Z|variable|Y|Y|Proxy|2|Planned|Build failures|Internal cache (Artifactory)|Engineering|DevEx|DEV-88|EGR-RULE-04|netpol-proxy-egress|DNS|Build job test|N|N|Mirror after cutover|
|Certs|LetsEncrypt ACME|acme-v02.api.letsencrypt.org|acme-v02.api.letsencrypt.org|Public IP|HTTPS|443|Egress|Add-on|ingress|cert-manager|cm:1.14.0|quay.io|Public|No|Firewall|EgressGW|Yes|Public|Token|Low|High|Periodic|daily (renew window)|2025-09-04T00:05Z|low|N|N|Keep Public via GW|3|Planned|Cert expiry|DNS-01 fallback|Platform|Platform|PLAT-321|EGR-RULE-05|netpol-egress-gw|DNS|Renewal dry-run|N|N|Migrate to DNS-01 later|

> Note: Replace sample row values with your actual data. Remove columns you choose not to track, or keep them empty until populated.

## 2. Full Column Set (Extended Sheet)
Use a CSV or spreadsheet with these headers:
```
Category,Service / Function,FQDN / Domain,Exact Host,IP,Protocol,Port(s),Direction,AKS Layer,Namespace,Workload,Container Image,Image Registry,Access Mode,Private Link Available,Current Path,Future Path,Requires DNS,DNS Zone,Auth Method,Data Sensitivity,Criticality,Usage Pattern,Avg Frequency,Last Seen,Volume,Fallback / Cache,Can Mirror/Proxy,Plan,Migration Phase,Status,Risk If Blocked,Mitigation,Owner (Business),Owner (Technical),Approval Ticket / Ref,Policy Rule ID,NetPol / Cilium Policy Ref,Logging Source,Verification Method,Verified,Decommission Candidate,Notes
```

### Column Definitions (Concise)
- **Category**: Group (Azure Core, Registry, Secrets, Packages, Monitoring, Certs, SaaS, Partner, Other).
- **Service / Function**: Human-readable purpose.
- **FQDN / Domain**: Base domain aggregated.
- **Exact Host**: Specific host if single (e.g. mykv.vault.azure.net).
- **IP**: Fixed / Private Endpoint IP if applicable.
- **Protocol / Port(s)**: Transport details.
- **Direction**: Egress / Ingress / Bi.
- **AKS Layer**: Platform, Add-on, Workload.
- **Namespace / Workload / Container Image / Image Registry**: Source identity.
- **Access Mode**: Public, PrivateLink, VNet Internal, Proxy, Mirror.
- **Private Link Available**: Y/N/Planned.
- **Current Path / Future Path**: e.g. Firewall → PrivateLink.
- **Requires DNS**: Whether domain resolution needed (some direct IP flows may not).
- **DNS Zone**: Public or Private (internal override?).
- **Auth Method**: MSI, AAD, Key, Token, None.
- **Data Sensitivity**: None–High; categorize data exchanged.
- **Criticality**: Blocker, High, Med, Low.
- **Usage Pattern**: Boot, Periodic, Burst, Continuous, Ad-hoc.
- **Avg Frequency / Last Seen / Volume**: For prioritization & validation.
- **Fallback / Cache**: Can operate offline via cache.
- **Can Mirror/Proxy**: Feasibility flag.
- **Plan**: Privatize, Proxy, Mirror, Keep Public, Replace, Remove.
- **Migration Phase / Status**: Track progress per phased plan.
- **Risk If Blocked / Mitigation**: For risk register linkage.
- **Owners**: Business + Technical accountability.
- **Approval Ticket / Ref**: Change or architecture review ID.
- **Policy Rule ID / NetPol Ref**: Once enforced, link to rule objects.
- **Logging Source**: Flow, DNS, App, None.
- **Verification Method / Verified**: How validation done (TestPod, Synthetic, Curl).
- **Decommission Candidate**: Y/N.
- **Notes**: Freeform.

## 3. Auxiliary Sheets
### registries.md
|Registry|PrivateLink|Mirror Strategy|Sync Tool|Frequency|Owner|Status|Notes|
|--------|-----------|---------------|--------|---------|-----|------|-----|
|docker.io|No|Local cache (Harbor)|crane sync|Daily|Platform|Planned|Rate limit mitigation|
|ghcr.io|No|Harbor proxy|oras mirror|Daily|Platform|Planned|Select base images only|
|quay.io|No|Harbor pull-through|Harbor|Weekly|Platform|Backlog|Filter required|

### package_repos.md
|Ecosystem|Primary Source|Mirror URL|Proxy?|Criticality|Plan|Status|Notes|
|---------|--------------|----------|------|----------|----|------|-----|
|npm|registry.npmjs.org|https://npm.internal.corp|Yes|High|Proxy|In Progress|Artifactory configured|
|PyPI|pypi.org|https://pypi.internal.corp|Yes|High|Proxy|Planned|Bandwidth spikes|
|Maven|repo1.maven.org|https://maven.internal.corp|Yes|High|Mirror|In Progress|Nexus staging|

### dns_rules.md
|Domain Pattern|Action|Justification|Owner|Ticket|Implemented|Notes|
|--------------|------|------------|-----|------|-----------|-----|
|*.vault.azure.net|Allow|Secrets retrieval|Security|SEC-123|Y|Private Link|
|*.ods.opinsights.azure.com|Allow|Log ingestion|Observability|OBS-45|Y|Private Link|
|*.docker.io|Allow|Base images|Platform|PLAT-77|N|Will mirror later|
|*.facebook.com|Deny|Not required|Security|SEC-200|Y|Baseline block|

### risk_register.md
|ID|Dependency Ref|Risk Description|Severity|Likelihood|Score|Mitigation|Owner|Target Date|Status|
|--|--------------|----------------|--------|----------|-----|---------|-----|-----------|------|
|R1|Key Vault|Secrets fetch blocked|5|2|10|Private Link + test|Security|2025-01-15|Planned|
|R2|Ubuntu Archive|Patch lag|4|3|12|Apt proxy + mirror|Infra|2024-12-01|In Progress|
|R3|ACME|Cert renewal failure|5|2|10|DNS-01 fallback|Platform|2025-02-01|Planned|

### test_canaries.md
|Dependency Ref|Test Type|Schedule|Success SLO|Last Result|Alert Channel|Notes|
|--------------|---------|--------|-----------|-----------|-------------|-----|
|ACR|Image pull|5m|99.9%|OK 2025-09-04T12:00Z|PagerDuty:Platform|-
|Key Vault|Secret read|5m|99.9%|OK 2025-09-04T12:00Z|PagerDuty:Security|-
|Log Analytics|HTTPS ingest|5m|99.5%|OK 2025-09-04T12:00Z|Slack #obs-alerts|-

## 4. Scoring & Prioritization
Risk Score = Severity (1–5) * Likelihood (1–5). Prioritize items with Score ≥12 before firewall removal. Mark critical dependencies (Criticality=Blocker) for earliest Private Link or proxy.

## 5. Data Collection Hints
- **Images**: `kubectl get pods -A -o jsonpath="{range .items[*]}{.spec.containers[*].image}{'\n'}{end}" | sort -u`
- **DNS Baseline**: Temporarily enable CoreDNS logging for 7 days (remove after to reduce noise).
- **Flow Logs**: Enable NSG Flow Logs v2; aggregate by destination; join with DNS queries.
- **Firewall Logs**: Export top domains (last 30d) to seed the sheet.
- **Ownership**: Auto-tag namespace owners from internal CMDB; manual verify high-risk entries.

## 6. Workflow
1. Populate sheet → categorize → assign owners.
2. Decide Plan & Phase for each dependency.
3. Implement Private Link / proxy / mirror.
4. Add NetworkPolicy / Cilium egress rule & DNS rule.
5. Run verification canary; mark Verified.
6. After all Blocker + High done: schedule firewall removal window.
7. Post-cut: watch logs for uncatalogued domains; update sheet.

## 7. Legend
- **Access Mode**: Public (direct public IP/FQDN), PrivateLink (PaaS private endpoint), VNet Internal (internal service), Proxy (via egress proxy), Mirror (internal mirror of external content).
- **Phase**: 0 Discovery, 1 Hardening, 2 Egress Shrink, 3 Cutover, 4 Optimize.
- **Status**: Unassessed, Planned, In Progress, Validated (tested but not cut), Cutover (live on new path), Deprecated (no longer needed).

## 8. Maintenance Guidance
- Weekly: Review new Flow/DNS entries not in sheet; decide plan.
- Monthly: Recalculate risk scores; escalate stalled Blockers.
- Quarterly: Prune Deprecated entries; archive sheet snapshot.

---
Feel free to adjust column breadth; you can collapse to the minimal table for executive reporting while keeping the full CSV for engineers.
