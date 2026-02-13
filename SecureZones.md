# Enterprise Security Zone Architecture & Management Infrastructure Strategy

## Executive Summary

This document defines a comprehensive security zone model for organizations spanning DevOps environments (unit test through production) and high-value isolated systems. It provides:

1. **Six-zone segmentation model** aligned to modern DevOps and security governance
2. **Risk-based architecture** with graduated security controls per tier
3. **Four architectural options** for security management infrastructure (SIEM, XDR, EDR, patching)
4. **Detailed pros/cons analysis** for each approach
5. **RBAC framework** and cross-account access controls
6. **Change management** and testing strategy for security tools

---

## Part 1: Security Zone Definitions & Model

### 1.1 Zone Hierarchy Overview

The security model implements **six logical zones** organized by risk level, data sensitivity, and change velocity. Each zone has distinct security requirements, access patterns, and tool deployments. [web:16][web:17][web:18][web:21][web:24]

The zones follow a **"promotion pipeline"** where code/infrastructure flows from lower tiers through production-like environments before reaching production or high-value systems. This design enables progressive validation while preventing "blast radius" escalation.

```
┌─────────────────────────────────────────────────────────────┐
│  ISOLATION & RISK GRADIENT                                 │
├─────────────────────────────────────────────────────────────┤
│  Zone 1: Unit Test          [LOW RISK, HIGH CHURN]         │
│           ↓ (CI/CD artifact promotion)                      │
│  Zone 2: Lower Non-Prod     [LOW RISK, MODERATE CHURN]     │
│           ↓ (deploy automation)                             │
│  Zone 3: P-like (Staging)   [MEDIUM RISK, LOW CHURN]       │
│           ↓ (blue-green production promotion)               │
│  Zone 4: Production         [HIGH RISK, LOW CHURN]         │
│           → (parallel)                                      │
│  Zone 5: External UAT       [HIGH RISK, MODERATE CHURN]    │
│           (internet-facing)                                 │
│                                                             │
│  Zone 6: High-Value/Air-Gap [CRITICAL RISK, MINIMAL CHURN] │
│           (isolated, separate management)                   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Detailed Zone Specifications

#### Zone 1: Unit Test
**Purpose**: Local/ephemeral developer sandboxes for unit, integration, and component testing.

**Characteristics**:
- Environment: Containers, local VMs, ephemeral cloud resources (destroyed after test)
- Data: Synthetic, anonymized, or test fixtures only—never production data
- Access: Individual developers, CI/CD pipeline agents
- Network: Fully isolated; no routing to other zones
- Lifespan: Hours to days; high create/destroy frequency

**Security Posture**:
- No persistent EDR or SIEM agents (too much churn)
- Static analysis and secret scanning in CI pipeline only
- Local container image scanning (trivy/grype)
- No inter-zone network access; firewall blocks inbound/outbound
- Minimal compliance auditing required

**Justification**: Optimized for developer velocity. Security controls at CI pipeline level (code scanning, dependencies). Cost of persistent monitoring exceeds risk given isolation and synthetic data.

**Key Risk**: Developers accidentally expose secrets in test data. **Mitigation**: Secret scanning (GitLeaks, TruffleHog) blocks commits; sanitized test fixtures auto-generated. [web:55]

---

#### Zone 2: Lower Non-Prod
**Purpose**: Feature/integration testing, early CI/CD validation environments (dev, QA, feature branches).

**Characteristics**:
- Environment: Cloud accounts/subscriptions (AWS/Azure/GCP dev/test); moderate resource lifetime (days to weeks)
- Data: Test data, sanitized copies of lower-risk production data; no PII or secrets
- Access: Developers, QA engineers, CI/CD pipelines; looser controls than Prod
- Network: Isolated from Prod; one-way artifact pulls from CI only
- Change frequency: High (multiple deployments daily)

**Security Posture**:
- Lightweight EDR (agent deployed but rules relaxed; baseline enabled, advanced hunting disabled)
- Basic SIEM log forwarding (apps + infra logs); no real-time correlation
- Patch management: Security patches only, ~30-day SLA (non-critical patches optional)
- Vulnerability scanning: Weekly; medium/high severities tracked informally
- NAC: Optional; basic device identity but no network enforcement
- WAF: Not required (no internet exposure); basic rate limiting if any external access

**Justification**: Best-effort security appropriate for test environments. Supports learning, experimentation, and fast iteration without overhead of production controls. Foundation for P-like promotion testing.

**Key Risk**: Misconfigured permissions allowing access to Prod secrets/creds. **Mitigation**: IAM policy-as-code enforces no cross-zone role assumption; secrets stored in vault with separate rotation schedule. [web:52]

---

#### Zone 3: P-like (Production-Like/Staging)
**Purpose**: Deployment staging environment that mirrors Production infrastructure, configurations, backups, monitoring, and security controls for end-to-end validation.

**Characteristics**:
- Environment: Cloud account/subscription identical to Prod (same VPC/VNET, AZs, scaling policies)
- Data: Sanitized production data (PII removed but schema/volume realistic); or synthetic high-volume data
- Access: SRE/DevOps, QA engineers, Security team; stricter than Lower Non-Prod but not as strict as Prod
- Network: Isolated from Prod; connected to shared management stack; one-way deployments
- Change frequency: Moderate (1-3 deployments daily); canary/blue-green testing

**Security Posture**:
- Full EDR suite (agents with production-grade rules, advanced hunting enabled)
- Production-grade SIEM (full data pipeline, real-time correlation, playbook testing)
- Patch management: Full schedule (critical: 7 days, standard: 30 days, aligned with Prod cadence)
- Vulnerability management: Daily scans; SLA-driven remediation (same as Prod)
- WAF: Enabled for web workloads; rule testing before Prod deployment
- Secrets management: Vault with access logging; rotation same as Prod
- Backup/recovery: Full snapshots, tested recovery procedures (RTO/RPO matching Prod)
- Monitoring/alerting: Identical dashboards to Prod (data only, no cross-pipeline alerts)

**Justification**: P-like is the **primary pre-production validation environment**. It validates security tools, agent behavior, and rule effectiveness before Prod/High-Value rollout. [web:16][web:17][web:21][web:24][web:49]

**Key Uses**:
1. Test SIEM/XDR rule changes before Prod deployment
2. Validate EDR agent/platform upgrades (staged rollout: 10% P-like → 50% P-like → 100% Prod)
3. Load-test WAF rules and DDoS protection
4. Verify patch cycles and automated remediation workflows
5. Disaster recovery testing (backup restore, failover)

**Key Risk**: Treating P-like as "safe sandbox" and running risky changes there, then promoting untested code to Prod. **Mitigation**: Enforce same approval/peer-review gates for P-like→Prod as Lower→P-like; treat P-like data with same confidentiality as Prod.

---

#### Zone 4: Production
**Purpose**: Live, revenue-generating customer-facing workloads and data.

**Characteristics**:
- Environment: Multi-region, multi-AZ cloud deployment (AWS/Azure/GCP prod accounts)
- Data: All customer/business-critical data (PII, transactions, secrets)
- Access: SRE/Ops (restricted), Security (read/alert-response), Developers (read-only for troubleshooting)
- Network: Internet-facing (with perimeter controls); restricted outbound
- Change frequency: Low (1-2 planned deployments per week; emergency patches as-needed)

**Security Posture** (Full Enterprise):
- Enterprise EDR/XDR (all hosts; advanced hunting, threat intelligence integration)
- Enterprise SIEM (centralized log aggregation; 24/7 SOC monitoring, real-time alerting)
- Patch management: Aggressive cadence (critical: 0-7 days, standard: 14-30 days)
- Vulnerability management: Continuous scanning; SLA-driven prioritization
- WAF/DDoS: Multi-layered (WAF + cloud DDoS + CDN protection)
- Secrets management: HSM-backed vault; rotation every 90 days; audit every access
- MFA: Mandatory for all admin access; hardware keys for sensitive roles
- Bastion/Jump hosts: Hardened, session-recorded; geo-fenced to your region
- Incident response: On-call SOC, playbooks, post-mortems
- Compliance: Audit logging, retention per regulatory requirement (SOC 2, PCI-DSS, ISO 27001)

**Justification**: Production is the highest-trust environment. Controls are mandatory and overhead is justified. Failure impacts customers, revenue, and reputation.

**Key Risk**: Accidental outage from overzealous patching/config change. **Mitigation**: Blue-green deployments, canary rollouts, automated rollback on alert spike.

---

#### Zone 5: External UAT
**Purpose**: User acceptance testing exposed to internet; acceptance criteria validation before Prod launch (beta customers, QA partners).

**Characteristics**:
- Environment: Cloud account separate from Prod; Prod-like infrastructure but short-lived (weeks to months)
- Data: Test data + limited production-like data (no sensitive customer PII); ephemeral
- Access: QA teams, beta customers, security testing teams; limited but internet-reachable
- Network: Internet-facing; DMZ architecture (WAF in front); restricted outbound to Prod/internal networks
- Change frequency: Moderate (daily to weekly changes for test scenarios)

**Security Posture**:
- Perimeter hardening: WAF (OWASP Top 10 + custom rules), DDoS mitigation, rate limiting
- EDR: Full agents (same as Prod) to detect intrusions or abuse
- SIEM: Separate partition in Prod-grade SIEM; alert on anomalous access patterns
- Patch management: Same as Prod cadence (proactive security)
- Vulnerability scanning: Weekly; fast remediation (external threats active)
- Secrets management: Vault with limited TTL credentials; no long-lived keys
- Network segmentation: No direct routes to Prod; one-way log forwarding only
- Access controls: Separate IAM; time-gated access for beta testers (auto-revoke post-UAT)
- Monitoring: Real-time alerts on suspicious activity; auto-blocking of repeated attacks

**Justification**: External UAT is "internet-facing production-grade" from security perspective. Must withstand attacks and provide realistic performance/security validation. Cannot share management with Prod due to attack surface.

**Key Risk**: Attacker escalation from UAT to Prod/High-Value via misconfigured network routes or compromised shared credentials. **Mitigation**: Network firewalls enforce strict segmentation; separate IAM accounts; no cross-zone assume-role. [web:47][web:51]

---

#### Zone 6: High-Value/Air-Gap
**Purpose**: Isolated environment for sensitive/critical assets (classified data, compliance-critical systems, strategic intellectual property, high-security workloads).

**Characteristics**:
- Environment: Dedicated infrastructure (physical or dedicated cloud tenant); separate cloud account/subscription
- Data: Highest sensitivity; PII, trade secrets, compliance-critical records
- Access: Dedicated team; no overlap with Prod/UAT ops; multi-approval for all changes
- Network: **Virtual or physical air-gap**: no direct routes to internet or other zones; one-way conduits only (for updates/logs/software delivery)
- Change frequency: Minimal (quarterly updates; emergency patches via offline approval)
- Management infrastructure: **Completely separate** from other zones

**Security Posture** (Maximum Protection):
- Dedicated SIEM/XDR/EDR stack (not shared with Prod stack)
- Separate IdP/identity system (AD forest or independent Okta org) or local accounts with air-gapped admin bastion
- Hardened management: Air-gapped jump host for admin access (no internet); removable media for updates
- Patch management: Offline testing; manual validation before deployment; rollback via offline procedure
- Secrets management: Air-gapped HSM or local vault; no cloud-based secret service
- Network: One-way conduits for log forwarding, signature updates, software delivery (unidirectional firewalls)
- Physical security: Surveillance, access logs, badge-gated data center sections
- Backup: Air-gapped backup storage; tested recovery offline
- Compliance: Enhanced audit logging, tamper-evident controls, executive review

**Virtual Air-Gap Implementation** (if cost is concern):
- Separate management stack + dedicated network security appliances
- One-way log forwarding (syslog) to external SIEM (read-only, no query-back)
- Tightly locked-down firewall rules (explicit allow-lists only)
- No bidirectional traffic except for approved maintenance windows
- Monthly network audit/pen-test of "conduits"

**True Physical Air-Gap Implementation**:
- No network connection to other zones
- Updates via offline media (USB drive, QR codes for configs)
- Staff trained in air-gapped operations
- Enhanced physical security
- Manual log review + offline analysis

**Justification**: High-Value assets warrant maximum protection. Regulatory requirements (e.g., HIPAA, classified data handling, financial compliance) may mandate air-gapping. Recovery from Prod compromise must be guaranteed. [web:6][web:12][web:50][web:53]

**Key Risk**: Operational friction leading to "side-stepping" the air-gap (e.g., admins taking shortcuts, connecting USB drives with malware). **Mitigation**: Clear operational procedures, regular training, automated controls (e.g., USB port disabling), surprise audits.

---

### 1.3 Zone Access Matrix (Allowed Data/Traffic Flows)

This matrix defines what flows between zones are **explicitly allowed** (all else denied by default). [web:44][web:45][web:46][web:51]

| From Zone | To Zone | Allowed Flow | Mechanism | Notes |
|-----------|---------|--------------|-----------|-------|
| Unit Test → Lower Non-Prod | Artifact push (container images, compiled binaries) | CI/CD pipeline (GitHub Actions, Jenkins) | Read-only artifacts; no shell access; signed/scanned |
| Lower Non-Prod → P-like | Application deployment (IaC, helm charts) | Automated pipeline with approval gates | Peer review required; rollback validated |
| P-like → Production | Production release (canary/blue-green) | CI/CD with dual approval (dev + SRE) | Monitoring for alert spike; auto-rollback enabled |
| Any → External UAT | Test artifact deployment | Controlled pipeline; separate IAM | Temporary access tokens (24hr TTL); no secrets |
| Any → High-Value | None (default-deny; virtual air-gap only) | Offline transfers or one-way conduits | Conduits: syslog-only, no reverse traffic; approved maintenance windows |
| Prod/P-like/Non-Prod → High-Value | Log forwarding (one-way, syslog) | Syslog/forward agent, no query-back | High-Value receives logs; cannot push commands back |
| Any → Any | Admin access (via Bastion) | Session manager (AWS SSM, Azure Bastion) | Geo-fenced, MFA required; logged; TTL 1-4 hours |
| All zones | Internet | Egress for package updates, API calls | Proxy/firewall outbound rules; no inbound |

**Default Deny Principle**: All inter-zone traffic blocked by firewall unless explicitly permitted above. [web:2][web:45][web:46][web:47][web:51]

---

## Part 2: Security Management Infrastructure (SIEM, XDR, Patch Mgmt)

### 2.1 Conceptual Overview

Organizations need **security management platforms** to monitor, detect, and respond to threats across all zones:

- **SIEM** (Security Information and Event Management): Centralized log aggregation, correlation, alerting
- **XDR/EDR** (Extended/Endpoint Detection & Response): Agent-based threat detection, incident response, forensics
- **Patch Management**: Vulnerability scanning, patch orchestration, compliance reporting
- **Configuration Management**: Baseline hardening, compliance scanning, drift detection
- **XOAR** (eXtended Orchestration & Automation Response): Cross-platform workflow automation, playbook execution

These can be deployed in **three architectural models**:
1. **Single shared stack** (all zones) – Lowest cost, single failure point
2. **Two stacks** (Prod-grade + High-Value isolated) – Balanced cost/isolation
3. **Three stacks** (Low-Sec + Prod-grade + High-Value) – Maximum isolation but highest cost

---

### 2.2 Option 1: Single Shared Enterprise Stack (All 6 Zones)

**Architecture**: One SIEM, one XDR/EDR platform, one patch mgmt system; all zones feed into unified infrastructure. Logical separation via tenants, workspaces, or policy sets.

**Deployment Details**:

- **SIEM**: Central instance (Splunk/ELK/Datadog/Azure Sentinel) with 6 separate workspaces or data partitions (one per zone)
- **Agents**: EDR agents deployed across all zones; rules vary by zone (relaxed for Unit/Lower, full for Prod/High-Value)
- **Log Flow**: All logs to central SIEM; retention per zone (Unit: 7 days, Lower: 30 days, P-like/Prod: 90 days, High-Value: 365+ days)
- **Bastion**: Single shared bastion for all admin access; role-based policies control zone access
- **Patch Mgmt**: One patching platform (Qualys, Rapid7, Tanium); separate collections/update windows per zone

**Pros**:
- Lowest licensing cost: Single SKU/license per tool
- Unified visibility: Easy to correlate threats across pipeline (e.g., malware in Unit → trend to Prod)
- Consistent baselines: Single set of detection rules, compliance policies; easier to tune
- Single pane of glass: SOC team has one console for all zones
- Simplified onboarding: New zones inherit existing templates
- P-like testing representative: Uses exact Prod SIEM, so rule testing is high-fidelity

**Cons**:
- **Critical SPOF**: Compromise of SIEM or bastion = compromise of all zones including High-Value
- **Non-compliant for High-Value**: Air-gap/regulatory requirements typically demand independent management
- **Noise overload**: Unit Test and Lower Non-Prod churn creates alert fatigue; hard to tune without impacting Prod signals
- **Shared blast radius**: Vulnerability in SIEM affects all zones; patch becomes mandatory for all
- **Scale pain**: Large SIEM ingestion at high volume; performance degrades with 6 zones if not properly partitioned
- **Identity collapse**: Single set of admin accounts/identities; higher risk of privilege creep or lateral movement
- **Approval complexity**: Hard to implement different change policies per zone (Prod strict, Unit lenient) in shared SIEM

**Cost Profile**: $$
- SIEM: 1 instance (large)
- XDR/EDR: 1 platform (10,000+ agents)
- Patch: 1 platform
- Total: ~Lowest absolute cost

**Compliance Fit**: Poor for regulated environments (HIPAA, PCI-DSS, classified). Fair for general enterprise.

**Recommendation**: Only suitable for **organizations with no air-gap/high-value zone** or those accepting moderate risk. Not recommended for financial services, healthcare, or defense contractors.

---

### 2.3 Option 2: Two Stacks (Prod-Grade for Lower Tiers + P-like/Prod/UAT; Dedicated for High-Value)

**Architecture**: Stack 1 (Enterprise-grade) manages Unit, Lower Non-Prod, P-like, Prod, External UAT with light agents for lower tiers and full for upper tiers. Stack 2 (dedicated) manages High-Value zone only.

**Deployment Details**:

**Stack 1 (Prod-Grade)**:
- SIEM: Enterprise instance (Splunk/ELK/Sentinel) with 5 workspaces (Unit, Lower, P-like, Prod, UAT)
- EDR: Full XDR platform; agents light on Unit/Lower, production-grade on P-like/Prod/UAT
- Patch: Shared patching platform; separate collections for each tier
- Bastion: Primary bastion for Zones 1-5; role-based access (Developers → 1-2, SRE → 3-4, SOC → all 5)

**Stack 2 (High-Value)**:
- SIEM: Dedicated instance (same vendor or different)
- EDR: Dedicated XDR/EDR platform; isolated agents/console
- Patch: Separate patching platform; offline or conduit-based
- Bastion: Air-gapped bastion; physical/logical isolation; multi-party approval for access

**Conduit between Stacks**:
- One-way syslog from Stack 1 to Stack 2 (High-Value SIEM receives selected Prod logs for correlation)
- No reverse traffic (High-Value cannot query/control Stack 1)
- Approved maintenance windows: HV platform updates pulled via secure file transfer

**Pros**:
- Strong High-Value isolation: Separate management means Prod compromise doesn't auto-cascade to HV
- Cost-effective: Only 2 stacks vs 3; High-Value gets dedicated protection without full duplication
- P-like still representative: Uses Stack 1 (same as Prod), so testing is high-fidelity
- Reasonable ops overhead: Main SOC team manages Stack 1; small dedicated team for Stack 2
- Compliance-friendly: Meets most air-gap/segregation requirements (NIST, ISO 27001)
- Graduated access controls: Can implement strict separation between Stacks without breaking lower-tier workflow
- Conduits auditable: One-way syslog easy to monitor/validate; no shell access paths

**Cons**:
- Dual maintenance: Two SIEM instances to patch/tune; rule drift risk if not carefully managed
- Partial visibility gap: High-Value data not available in Stack 1 console; requires separate login to correlate
- Conduit design required: Must implement/maintain one-way forwarding, network rules; adds complexity
- Higher license cost than Option 1: ~1.5x licensing (SIEM license scales with data ingestion)
- Identity coordination: Stack 1 and Stack 2 have separate IdPs; admins manage two identity domains
- Operational procedures: Different playbooks, escalation paths for Stack 1 vs Stack 2 incidents

**Cost Profile**: $$ (1.5x Option 1)
- SIEM Stack 1: 1 large instance
- SIEM Stack 2: 1 medium instance (lower volume)
- XDR/EDR: 2 platforms (8,000 agents in Stack 1, 500 in Stack 2)
- Patch: 2 platforms
- Conduit infrastructure: Additional firewall rules, log forwarding agents
- Total: ~1.5x Option 1

**Compliance Fit**: Good for regulated environments with air-gap zones. Meets NIST 800-53 segregation requirements.

**Recommendation**: **RECOMMENDED for most enterprises with high-value/air-gap zones**. Best balance of cost, usability, and isolation.

---

### 2.4 Option 3: Three Independent Stacks (Low-Sec + Prod-Grade + High-Value)

**Architecture**: Three completely independent security management stacks, optionally unified via XOAR at the top for orchestration.

**Deployment Details**:

**Stack 1 (Low-Sec Stack – Unit + Lower Non-Prod)**:
- SIEM: Smaller, cost-effective instance (Open Source ELK, Wazuh, or budget tier Splunk)
- EDR: Lightweight agent (osquery, Wazuh EDR) or basic endpoint tools
- Patch: Simple patching tool (Patchman, Landscape.io) or infrastructure-as-code patching
- Retention: Short (7-30 days); best-effort monitoring
- Team: 0.5 FTE managing

**Stack 2 (Prod-Grade Stack – P-like + Prod + External UAT)**:
- SIEM: Enterprise instance (Splunk/ELK/Sentinel/Datadog)
- EDR: Full XDR/EDR platform (CrowdStrike, Sentinel One, Microsoft Defender)
- Patch: Enterprise platform (Qualys, Rapid7, Tanium)
- Retention: 90+ days; real-time monitoring
- Team: 2-3 FTE SOC team

**Stack 3 (High-Value Stack)**:
- SIEM: Dedicated instance (often different vendor or air-gapped instance)
- EDR: Dedicated platform; may use different vendor than Stack 2 for diversity
- Patch: Air-gapped or conduit-based; manual validation
- Retention: 365+ days; compliance-driven
- Team: 1 FTE dedicated + on-call

**XOAR Orchestration Layer** (Optional):
- Central workflow engine (Demisto/Cortex XSOAR, ServiceNow) that can trigger playbooks across all three stacks
- Aggregates alerts from 3 SIEMs for risk scoring
- Auto-escalation: Low-Sec alert → Prod-Grade investigation → High-Value threat review
- Caution: XOAR becomes SPOF if it has write-back to all stacks

**Pros**:
- **Maximum isolation**: Each stack is independent; compromise doesn't cascade
- **Tailored optimization**: Low-Sec can run experimental tools; Prod-Grade focused on accuracy; High-Value on compliance
- **Flexible scaling**: Can outsource Low-Sec to MSSP or third-party; keep Prod/HV in-house
- **Experimentation freedom**: Test new SIEM features, alert rules, or EDR policies in Low-Sec without risk
- **Parallel deployments**: Prod-Grade and High-Value can patch on different schedules
- **Strong change testing**: Test in Low-Sec → stage in Prod-Grade P-like subset → full Prod-Grade → High-Value canary
- **Regulatory compliance**: Clean separation for multi-tenancy, HIPAA, classified environments
- **Diverse vendors**: Can use different SIEM vendors per stack (hedge against product-specific vulns)

**Cons**:
- **Highest TCO**: 3x licensing, 3x infra costs, 3x patch/upgrade overhead
- **Fragmented visibility**: Analysts must log into 3 consoles; harder to correlate cross-stack threats
- **XOAR complexity**: Additional layer to manage; if XOAR is compromised, all 3 stacks at risk
- **Rule proliferation**: Same detection rules duplicated across 3 SIEMs; drift and inconsistency over time
- **People overhead**: Requires strong team (5-8 FTE) to maintain 3 stacks; scaling challenge for smaller orgs
- **Integration pain**: More APIs, more failure points, more troubleshooting
- **Onboarding**: Steeper learning curve for new analysts; different console per stack

**Cost Profile**: $$$ (3x Option 1)
- SIEM Stack 1: 1 small/open-source instance
- SIEM Stack 2: 1 large enterprise instance
- SIEM Stack 3: 1 medium instance (air-gapped)
- XDR/EDR: 3 platforms (mix of agents across stacks)
- Patch: 3 platforms
- XOAR: 1 orchestration platform license
- Total: ~3x Option 1; ~2x Option 2

**Compliance Fit**: Excellent for highly regulated (HIPAA, PCI-DSS, DoD, classified) or multi-tenant environments.

**Recommendation**: For **large enterprises (500+ employees) or highly regulated industries** with dedicated security teams. Not recommended for startups or small orgs (over-engineered).

---

### 2.5 Option 4: Two/Three Stacks with Virtual Air-Gap for High-Value

**Architecture**: Builds on Option 2 or 3, but High-Value stack uses **virtual air-gap** via tightly controlled one-way conduits instead of full physical disconnection.

**Deployment Details**:

**Virtual Air-Gap Conduit Design**:
- **Unidirectional firewall** (e.g., Watergate, custom iptables) from Prod-Grade to High-Value
  - Whitelist: syslog (514/UDP), HTTPS for signature pull (443), NTP (123)
  - Everything else: blocked
  - Reverse traffic: DROPped (not rejected), no ICMP response
  
- **Outbound conduit** from High-Value to external (for updates/patches):
  - Cached CDN mirror or offline media for software delivery
  - Signed packages only; cryptographic verification
  - Monthly update windows
  
- **Monitoring of conduits**:
  - IDS rules on conduit firewall
  - Alerts on unexpected traffic patterns
  - Quarterly penetration testing of conduit

**Practical Example**:
```
Prod Stack (Stack 2)  →  Unidirectional Gateway  →  High-Value Stack (Stack 3)
   SIEM/XDR              Firewall/IPS/IDS           Isolated SIEM/XDR
   (monitored via)       (logs suspicious traffic)  (receives logs only)
   Flow rules:
   - syslog allowed (prod events → HV SIEM)
   - HTTPS blocked (HV cannot query prod SIEM)
   - SSH blocked (no remote command execution)
   - Reverse traffic: DROPped (hard break)
```

**Pros**:
- **Practical isolation**: Provides air-gap benefits (~95% protection) without full operational friction
- **Usable telemetry**: High-Value still receives real-time logs from Prod for correlation
- **Faster patching**: Can use online/conduit distribution instead of offline USB
- **Cost-effective**: Cheaper than full physical air-gap (no dedicated air-gapped infrastructure)
- **Testing fidelity**: P-like can simulate conduit-like behaviors (unidirectional logging, restricted updates)
- **Regulatory appeal**: Meets most air-gap intent requirements (NIST, ISO) with lower operational cost

**Cons**:
- **Conduit misconfiguration risk**: Single mistake in firewall rules can erode air-gap (e.g., ICMP reply leaks)
- **Not true air-gap**: Persistent/determined attacker may find bidirectional pivot if conduit isn't perfect
- **Maintenance burden**: Firewall rules must be audited monthly; rules drift over time
- **Inherited from parent**: Still subject to Option 2 or 3 limitations (dual/triple stacks, multiple teams)
- **Monitoring complexity**: Requires IDS/monitoring on conduit itself; adds cost/expertise

**Cost Profile**: $$ (Option 2 cost) to $$$ (Option 3 cost)
- Same as parent option + unidirectional firewall appliance (~$10-50k)
- Additional IDS/monitoring agents
- Total: Option 2 + 10-15% (cheapest air-gap option)

**Compliance Fit**: Good for most regulated environments. May not satisfy "true air-gap" definitions in DoD/classified contexts.

**Recommendation**: **BEST COMPROMISE** for organizations needing air-gap benefits without extreme operational friction. Widely recommended by CISO/architecture teams.

---

### 2.6 Testing and Change Management Strategy

Regardless of option chosen, a robust change management process for security tools is critical. [web:21][web:24][web:49][web:52]

#### Test Environment for Security Changes

You need at least **one dedicated pre-production segment** to safely test:
- Agent version upgrades (EDR/XDR, vulnerability scanner)
- SIEM rule/detection logic changes
- Patch management policy updates
- Platform upgrades (SIEM server, XDR console)
- Firewall/WAF rule changes

**Test Environment Tiers**:

| Environment | Used For | Size | Duration | Access |
|-------------|----------|------|----------|--------|
| **P-like Canary Subset** (10-20 servers) | Early validation of agent/rule changes | Small cluster | Permanent | SRE + Security team |
| **Feature branch with mocks** | Developer testing of new detection logic | Local/container | Hours | Individual analysts |
| **Lower Non-Prod sandbox** | Full rule/integration testing; CI scans | Medium | Days-weeks | QA + Security |
| **Offline test lab** (for HV only) | Air-gapped platform upgrades; patch testing | Single-node | Permanent | HV admin team |

#### Change Promotion Process

```
Development (Local)
    ↓ (rule code review + approval)
Feature Branch Testing (Lower Non-Prod)
    ↓ (unit tests pass; no false positives on known benign traffic)
P-like Canary (10% of P-like servers)
    ↓ (monitor for 3-7 days; low false-positive rate confirmed)
P-like Staging (50% of P-like servers)
    ↓ (monitor for 7 days; production-like volume; validated ROE)
P-like Full (100% of P-like servers)
    ↓ (final approval: CISO + product owner sign-off)
Production (Blue-green; monitor for alert spike)
    ↓ (auto-rollback if >2x false-positive spike)
High-Value (Off-hours; second approval layer)
```

#### Specific Change Scenarios

**Scenario 1: SIEM Rule Update**
- Write rule in dev; test in offline lab (Low-Sec SIEM test instance)
- Deploy to P-like SIEM (5% of data); tune threshold
- Deploy to full P-like (100% of data); monitor 7 days
- Deploy to Prod SIEM (100% of data); auto-disable if false-positive spike
- Document ROE (rules of engagement) in incident runbook
- Manual deployment to High-Value (if HV stack exists); offline validation first

**Scenario 2: EDR Agent Upgrade**
- Download agent to Lower Non-Prod; canary 5 hosts; monitor for 3 days (CPU, memory, compatibility)
- Canary 50% of P-like; monitor 5 days
- Canary 100% of P-like; 10 days monitoring
- Full Prod roll-out (canary deployment: 5% → 50% → 100% over 3 weeks)
- High-Value upgrade: offline testing in air-gapped lab first; manual deployment

**Scenario 3: Platform Upgrade (e.g., SIEM server)**
- Test upgrade in Lower Non-Sec stack first (low risk if downtime)
- Test in P-like during maintenance window (blue-green: old instance stays live)
- Execute in Prod during scheduled window (tested runbook, rollback plan)
- For High-Value: offline testing; air-gapped backup restore validation; on-site team present

#### Do You Need Another Environment?

**Short answer: No, but a dedicated P-like canary subset is essential.**

- **P-like Canary**: Dedicate 10-20 servers in P-like as a "security testing zone"; same stack as Prod but isolated via security groups/NSGs. This is your primary pre-prod validation.
- **Lower Non-Prod Lab**: Use existing Lower Non-Prod as your full-scale test bed (cheap, isolated, can break safely).
- **Offline Lab for HV**: If High-Value zone exists, maintain a small air-gapped lab that mirrors HV platform stack for offline testing (platform upgrades, backup restore procedures).

**You do NOT need a separate "security staging" environment**; P-like already serves that role.

---

## Part 3: Architecture Comparison & Recommendation

### 3.1 Comprehensive Option Comparison Table

| Aspect | **Option 1: Single Shared Stack** | **Option 2: Two Stacks (Recommended)** | **Option 3: Three Stacks** | **Option 4: Virtual Air-Gap** |
|--------|------|------|------|------|
| **Total distinct security stacks** | 1 | 2 | 3 (+XOAR) | 2-3 |
| **Total infrastructure zones** | 6 (Unit, Lower, P-like, Prod, UAT, HV) | 6 (or 7 with HV canary) | 6-8 (per-stack canaries) | Same as parent (2 or 3) |
| **SIEM instances** | 1 large | 2 (1 large + 1 med) | 3 (1 small + 1 large + 1 med) | 2-3 (same as parent) |
| **EDR/XDR platforms** | 1 | 2 | 3 | 2-3 |
| **Patch management** | 1 | 2 | 3 | 2-3 |
| **High-Value isolation** | Virtual (shared mgmt) | Strong (separate stack) | Maximum (dedicated stack) | Strong (virtual air-gap conduits) |
| **Cost (licensing + infra)** | $ (baseline) | $$ (1.5x) | $$$ (3x) | $$ (1.5x) to $$$ (3x) + 10% |
| **SOC team size (FTE)** | 1-2 | 2-3 | 5-8 | 2-4 |
| **Visibility (cross-zone correlation)** | Excellent (single pane) | Good (requires switching) | Fair (3 consoles) | Good (Stack 1 primary, HV read-only) |
| **P-like testing fidelity** | Perfect (exact Prod stack) | Perfect (exact Prod stack) | Perfect (exact Prod stack) | Perfect (exact Prod stack) |
| **Rule/policy duplication** | None | Minimal (some drift) | High (3 independent sets) | Minimal (one main stack) |
| **Single point of failure (SPOF)** | CRITICAL (SIEM/bastion = all zones) | Medium (SIEM/bastion for Stack 1; separate for Stack 2) | Low (3 independent stacks; XOAR is SPOF if write-back enabled) | Medium (conduits + parent SPOF) |
| **Compliance fit** | Poor (HIPAA, PCI, classified) | Good (NIST, ISO, air-gap) | Excellent (DoD, multi-tenant) | Good (most regulations) |
| **Experimental freedom** | Low (single SIEM; risky changes affect Prod) | Low (must test in Lower, not Prod stack) | High (Low-Sec stack can run experimental tools) | Low (limited to Lower Non-Prod) |
| **Operational procedures** | Simple (one playbook set) | Moderate (two playbook sets) | Complex (three playbook sets + XOAR) | Moderate (two-three with conduit mgmt) |
| **Scaling to new zones** | Easy (add to existing workspace) | Moderate (decide which stack) | Hard (add new stack or grow existing) | Moderate (add to parent stack or create conduit) |
| **Vendor lock-in risk** | High (single platform) | Medium (can use different vendors per stack) | Low (diversity possible per stack) | Medium (depends on parent) |
| **Incident response speed** | Fast (single SIEM; all data accessible) | Medium (may need cross-stack investigation) | Slow (pivot between 3 consoles; XOAR lag) | Fast (Stack 1 primary; HV escalation manual) |
| **Time to compliance audit** | Fast (single console; easy reporting) | Medium (two audit trails; more complex) | Slow (three audit trails; complex reconciliation) | Medium (plus conduit audit) |
| **Maintenance burden (patching, tuning, upgrades)** | Low (1 platform) | Medium (2 platforms) | High (3 platforms) | Medium (plus conduit rules) |

---

### 3.2 Decision Matrix: Which Option for Your Organization?

Use this to select the right option based on your constraints and requirements:

| Constraint / Requirement | Implication | Recommended Option |
|---------|-----------|--------|
| **Small org (20-50 people)**; no compliance requirements; no air-gap zone | Cost/complexity priority | **Option 1** (if no HV zone) → upgrade to **Option 2** if High-Value added |
| **Mid-market (50-200 people)**; regulated (e.g., financial, healthcare); requires High-Value zone | Balance of cost and isolation | **Option 2** (primary recommendation) |
| **Large enterprise (500+ people)**; highly regulated (HIPAA, PCI-DSS); multi-tenant or classified | Isolation + compliance priority | **Option 3** or **Option 4** |
| **Government/Defense contractor**; classified data; true air-gap requirement | Regulatory mandate | **Option 4** (virtual) or custom air-gapped setup |
| **Cost-constrained but needs air-gap** | Practical middle ground | **Option 4** (virtual air-gap on Option 2 base) |
| **Security-first culture; aggressive threat model** | Assume breach mentality | **Option 3** (maximum containment) |
| **DevOps-heavy; frequent deployments; high blast-radius concern** | Compartmentalization | **Option 3** (per-stack testing freedom) or **Option 2** (if cost-prohibitive) |
| **Audit/compliance-driven (SOC 2, ISO 27001, NIST)** | Clear segregation | **Option 2** (minimal extra overhead) |
| **CISO wants to "make everything air-gapped"** | Regulatory hedge | **Option 4** (virtual air-gap as compromise) |
| **Team wants simplicity; limited expertise** | Operational ease | **Option 1** (if no HV) or **Option 2** (sacrifice simplicity for isolation) |

---

### 3.3 Final Recommendation

**For most enterprises: Use Option 2 (Two Stacks) with the following specifics**:

**Stack 1 (Prod-Grade)**: Manages Unit Test, Lower Non-Prod, P-like, Production, External UAT
- SIEM: Enterprise tier (Splunk/ELK/Datadog/Azure Sentinel)
- EDR/XDR: Full platform (CrowdStrike, Sentinel One, Microsoft Defender)
- Patch: Enterprise scanner (Qualys, Rapid7, Tanium)
- Bastion: Primary admin jump host; role-based access per zone
- Team: 2-3 FTE (SOC analysts + manager)

**Stack 2 (High-Value Dedicated)**: Manages High-Value/Air-Gap zone
- SIEM: Same vendor as Stack 1 or air-gapped separate instance
- EDR/XDR: Dedicated agents/console; isolated from Stack 1
- Patch: Conduit-based or offline delivery
- Bastion: Air-gapped (no internet connection); multi-party approval for access
- Team: 1 FTE + on-call

**Conduit**: One-way syslog from Stack 1 to Stack 2; unidirectional firewall; audited monthly

**Testing Strategy**:
- P-like Canary: 10-20 servers in P-like for early validation of agent/rule changes
- Lower Non-Prod Lab: Full-scale pre-prod testing of security policies
- Offline HV Lab: Small air-gapped environment for HV platform upgrades
- Change process: Dev → Feature branch (Lower) → P-like canary → P-like full → Prod → HV

**Why Option 2**:
✅ Isolates High-Value zone (meets NIST/ISO/regulatory requirements)
✅ P-like remains perfect Prod mirror (high test fidelity)
✅ Reasonable cost (~1.5x Option 1)
✅ Manageable team size (2-4 people total)
✅ Good compliance posture without over-engineering
✅ Scalable to Option 3/4 if requirements evolve

---

## Part 4: RBAC and Cross-Account Access Control

### 4.1 Identity Model

Implement **cloud-native IAM** (AWS IAM, Azure RBAC, GCP IAM) with **central IdP** (Okta, Entra ID, PingIdentity) for federation.

**Principles**:
- **Zero standing credentials**: No long-lived access keys; use temporary assume-role/short-lived tokens
- **Just-in-Time (JIT) elevation**: Access granted for 1-4 hours; auto-revoke post-use
- **Least privilege**: Roles scoped to minimum required for function + zone
- **Audit everything**: Every assume-role, every access, logged to SIEM
- **Separate identities**: Production admins ≠ Unit Test developers ≠ High-Value specialists

**IdP Setup**:
- Central Okta/Entra tenant; multi-factor authentication (MFA) mandatory
- Device trust: Only managed devices (MDM/Intune) can access Prod
- Geographic fencing: Access denied if not in Hong Kong (or approved locations)
- Risk scoring: Anomalous access (odd time, device, location) triggers step-up auth

### 4.2 Core RBAC Tiers and Privileges

| Role Tier | Zone Scope | Permissions | Session Duration | MFA | Device Trust | Approval Required |
|-----------|-----------|----------|------------------|-----|--------------|-------------------|
| **Developer** | Zone 1-2 (Unit + Lower) | Read/write own namespaces; CI/CD trigger; view logs | 8 hours | Yes | Any | PR review |
| **Tester/QA** | Zone 2-5 (Lower, P-like, Prod read, UAT) | Deploy test scenarios; load test; read Prod logs (no modify) | 4 hours | Yes | Managed device | Test plan approval |
| **SRE/DevOps** | Zone 3-5 (P-like, Prod, UAT) | Infra scaling, restart, patch; view SIEM | 2 hours | Yes | Managed device | Peer review (for Prod changes) |
| **Security/SOC** | All zones (read), Prod-grade + HV (write) | View SIEM/logs all zones; tune rules Prod-grade + HV; IR access | 1 hour per zone | Yes | Managed device | None (within scope) |
| **High-Value Admin** | Zone 6 (HV only) | Full management; air-gapped ops | 30 minutes per session | Yes (hardware key) | Air-gapped bastion | Multi-party approval (3+ people) |
| **Platform/CI/CD** | Service account; zones 1-5 | Artifact push/deploy; read-only; no shell | Scoped token (15 min) | N/A (OIDC) | N/A | OIDC federation |
| **Break-Glass** | All zones (emergency) | Full emergency access | 4 hours (auto-disable) | Yes + approval | Any | 3+ approvers + geo-alert |

### 4.3 Detailed Cross-Zone Access Rules

#### Rule 1: No Backward Access (Lower → Higher)

**Policy**: Users/services in lower tiers CANNOT access higher tiers.

- Developers (Unit/Lower) cannot assume Prod or HV roles
- Lower Non-Prod runners cannot deploy to Prod directly
- Exception: CI/CD service accounts with explicit scoped permissions

**Implementation**:
```
IAM Policy:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::prod-account:role/SRE",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxx"
        },
        "IpAddress": {
          "aws:SourceIp": [
            "10.1.0.0/16"   # SRE office network only
          ]
        },
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

#### Rule 2: Forward Promotion Only (via CI/CD)

**Policy**: Code/artifacts flow forward through the pipeline; no rollback or bypass.

| Promotion | Mechanism | Approval |
|-----------|-----------|----------|
| Unit → Lower | Git branch merge + CI pipeline | PR review (1 approver) |
| Lower → P-like | Helm/Terraform + GitOps | Automated (pre-approved config) |
| P-like → Prod | Manual or auto (after P-like monitoring) | Dual approval (dev + SRE) |
| Any → UAT | Deployment pipeline + temporary creds | Test owner sign-off |
| Any → HV | Manual offline transfer (signed artifacts) | CISO approval |

#### Rule 3: Admin Access via Bastion Only

**Policy**: No direct shell access to Prod/HV; all access goes through hardened bastion with session recording.

**Architecture**:
```
Developer Workstation (MFA) 
    → IAM STS AssumeRole (1hr TTL) 
    → Bastion Host (SSH/RDP logged) 
    → Prod/HV target system (audited)
```

**Bastion Configuration**:
- Runs hardened OS (CIS benchmark)
- Session recording to S3 (encrypted)
- Time-limited credentials (1hr assume-role renewal)
- Alerts on unusual access (off-hours, failed attempts, lateral movement)
- No copy-paste (some implementations)
- MFA re-validation per session

#### Rule 4: Service Account Scoping

**Policy**: CI/CD and automation service accounts have minimal, time-limited permissions.

**Example: GitHub Actions deploying to P-like**:
```yaml
# GitHub Actions workflow
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Enable OIDC token
    steps:
      - name: Assume Role (P-like)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::plike-account:role/GitHubActionsRole
          aws-region: ap-southeast-1
          role-session-name: github-actions-${{ github.run_id }}
          role-duration-seconds: 900  # 15 minutes max
```

**Constraints**:
- Service account can ONLY deploy to P-like (no Prod access)
- Token expires after 15 minutes (auto-fail if slow)
- Artifact must be signed/scanned before assumption
- IAM policy restricts to: `s3:GetObject` (artifacts), `ecr:PutImage` (deploy), no delete/destroy

#### Rule 5: High-Value Access (Multi-Party Approval)

**Policy**: Access to High-Value zone requires 3+ people + business justification + audit trail.

**Process**:
1. Requester fills form: "I need HV access for X reason until date Y"
2. Manager approves
3. CISO approves
4. System grants 30-min access; MFA re-prompted
5. All actions logged; post-access debrief with security team

**Implementation** (using AWS Approval Workflow):
```
aws iam put-role-policy --role-name BreakGlass --policy-name BG-Policy \
  --policy-document '{
    "Statement": [{
      "Effect": "Allow",
      "Action": ["sts:AssumeRole"],
      "Resource": "arn:aws:iam::hv-account:role/HVAdmin",
      "Condition": {
        "StringEquals": {
          "aws:userid": "AIDAI23HZ27SI6FQMGNQ2"  # Specific person
        },
        "DateGreaterThan": {
          "aws:CurrentTime": "2026-02-13T00:00:00Z"
        },
        "DateLessThan": {
          "aws:CurrentTime": "2026-02-13T00:30:00Z"  # 30-min window
        }
      }
    }]
  }'
```

#### Rule 6: Observability (No Reverse Query)

**Policy**: Lower tiers can forward logs to upper tier SIEMs, but upper tiers cannot query back or execute commands in lower tiers.

**Implementation**:
- One-way syslog from Lower → Prod SIEM (no SSH/API access to Lower from Prod console)
- High-Value SIEM receives read-only copy of Prod logs (can search, but no remote execution)
- Firewall rules enforce unidirectional traffic

### 4.4 Audit and Monitoring

All access must be logged and monitored for anomalies:

| Event | SIEM Alert | Response | Escalation |
|-------|-----------|----------|-----------|
| Failed MFA (>3 attempts) | Alert SOC | Disable account temporarily; email user | Security team investigation |
| Access outside business hours | Alert SOC | Review reason; log for audit | If repeated, escalate to manager |
| Cross-account assume-role from unusual IP | Alert SOC | Requires immediate re-auth | Block if malicious |
| Prod access from non-managed device | Block access | Require MDM enrollment | User cannot proceed |
| Break-glass activation | Email CISO + SOC + Mgmt | Real-time monitoring of session | Auto-disable post-use; review |
| Data exfiltration attempt (large S3 copy) | Alert SOC | Block transfer; disable temp credentials | Investigate as potential breach |

---

## Conclusion

This document provides a **complete framework** for organizing your infrastructure into secure zones and selecting an appropriate security management architecture:

1. **Six-zone model** separates concerns (Unit → Lower → P-like → Prod/UAT ← High-Value)
2. **Option 2 (Two Stacks)** is the recommended balanced approach for most enterprises
3. **P-like environment** serves as the primary testing/validation ground for security changes
4. **RBAC and JIT access** enforce least-privilege principles across all zones
5. **Auditing and monitoring** track all access for compliance and incident response

Adapt this framework to your specific risk tolerance, compliance requirements, team size, and budget constraints.

---

## References

[web:2] – ITSG-38 Network Security Zoning
[web:6] – Air Gap Security Guide
[web:9] – Air Gap Best Practices (SentinelOne)
[web:12] – Network Segregation Guide
[web:15] – Air Gap Strategies
[web:16] – Dev/QA/Prod Environment Hierarchy
[web:17] – Microsoft Cloud Adoption Framework – Environments
[web:18] – AWS Choosing Git Branch Approach
[web:20] – Reddit: Dev/Staging/Production Discussion
[web:21] – PreProd Environment Guide
[web:22] – SIEM for DevOps
[web:24] – Securing Staging Environments
[web:25] – Reddit: 1 Company, 2 SIEMs
[web:27] – Dev, Test, Prod Best Practices 2025
[web:30] – Security Controls for Code Promotion
[web:31] – Managing Multiple Dev/Staging Environments
[web:33] – Security Within Dev/Staging/Production
[web:34] – DevOps Security Best Practices (JIT, MFA, Audit)
[web:35] – EDR: The Complete Guide
[web:36] – Kirkpatrick Price: Security by Environment
[web:39] – DevOps: Managing Multiple Environments
[web:40] – DevOps Security Best Practices 2025 (Break-Glass, RBAC)
[web:43] – 15 DevSecOps Best Practices
[web:44] – Palo Alto Networks Zone Protection
[web:45] – Alkira: Network Segmentation
[web:46] – Tufin: Enterprise Network Segmentation
[web:47] – Microsoft: Architecture Strategies for Segmentation
[web:48] – Cisco: Industrial Automation Security Design
[web:49] – DevOps Governance and Compliance
[web:50] – Fortinet: Air Gap Security Guide
[web:51] – Zero Networks: Segmentation & Microsegmentation
[web:52] – Policy as Code in DevOps Pipelines
[web:53] – Claroty: Air-Gapped Federal Infrastructure Protection
[web:54] – Tigera: NIST Security Zones
[web:55] – Harness: Ephemeral Environments Governance
[web:56] – Tufin: Segmentation vs Segregation
[web:57] – KPMG: DevSecOps Governance & Observability
