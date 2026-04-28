## Plan: Public Secure Service Bus Baseline Pattern

Design a Standard-tier Azure Service Bus reference architecture that keeps public connectivity enabled but tightly restricted through IP/network controls, Entra ID-only authentication, least-privilege RBAC, centralized logging, and policy enforcement mapped to the Azure Service Bus security baseline. The approach prioritizes secure-by-default guardrails and operational governance so public exposure remains controlled and auditable.

**Steps**
1. Define target architecture scope and trust boundaries: one Azure Service Bus namespace with one-to-many queues as messaging nodes, plus approved ingress source IP ranges from AKS outbound egress points (private AKS VNet -> Standard Load Balancer outbound public IP).
2. Phase 1 - Network Security Foundation:
3. Configure namespace networking for public access enabled with selected networks only; set default action deny; explicitly allow approved IPv4 CIDRs; document trusted-services usage as exception-only.
4. Add AKS egress controls so outbound identity is stable: attach a user-managed static public IP to AKS Standard Load Balancer outbound config, confirm effective outbound IP, and govern changes through CAB/pipeline approvals. This prevents firewall drift and access outages. *depends on 2*
5. Phase 2 - Identity and Authorization:
6. Enforce Microsoft Entra ID-only access by disabling local/SAS authentication on the namespace.
7. Use managed identities for all producer/consumer workloads; assign Azure Service Bus Data Sender/Data Receiver roles at queue scope per workload for each queue, and reserve namespace-scope assignment for platform operators only. *parallel with 6*
8. Define privileged access model for operators: separate admin roles, just-in-time/PIM activation, and break-glass process with audit requirements. *depends on 6*
9. Phase 3 - Data Protection and Transport:
10. Enforce TLS minimum version 1.2+ for namespace connectivity; validate client SDK compatibility and cipher posture.
11. Confirm encryption at rest with platform-managed keys (Standard tier constraint); document CMK as out-of-scope unless Premium migration occurs. *depends on 10*
12. Phase 4 - Monitoring, Detection, and Compliance:
13. Enable diagnostic settings for Service Bus resource logs and metrics to Log Analytics; route high-value signals to SIEM/Sentinel where used.
14. Define alerting for auth failures, network-rule denials, unusual sender/receiver patterns, and configuration changes on namespace/network settings.
15. Apply Azure Policy initiatives for Service Bus security posture (audit/deny/deployIfNotExists as applicable): local auth disabled, minimum TLS, diagnostic settings required, approved SKU/regions, and tagging standards. *parallel with 13*
16. Enable Defender for Cloud regulatory/compliance tracking and set remediation workflow for noncompliant controls. *depends on 15*
17. Phase 5 - Operational Runbook and Design Artifacts:
18. Produce architecture artifacts: logical diagram, control-to-baseline mapping matrix, RBAC matrix, and operational runbooks (IP allow-list changes, incident response, periodic access reviews).
19. Define lifecycle controls: quarterly role recertification, monthly firewall allow-list review, and baseline drift checks via policy compliance reports. *depends on 18*
20. Publish a decision log with explicit trade-offs of Standard vs Premium and trigger conditions for Premium migration (Private Endpoint/CMK/regulatory needs). *depends on 18*

**Relevant files**
- No existing repository files were found; this plan is greenfield and can be implemented by creating architecture documentation and IaC assets in this workspace.

**Verification**
1. Connectivity validation: the AKS outbound static public IP can connect; non-approved public IPs are denied.
2. Authentication validation: attempts using SAS/local auth fail; Entra ID token-based clients with managed identity succeed.
3. Authorization validation: sender identities cannot receive, receiver identities cannot send, and least-privilege scope boundaries are enforced.
4. Security configuration validation: namespace shows minimum TLS 1.2+, local auth disabled, selected networks mode active, and explicit allow-list entries present.
5. Observability validation: Service Bus logs/metrics arrive in Log Analytics; alerts trigger on synthetic failed auth and blocked network tests.
6. Compliance validation: Azure Policy compliance state is green for required controls; documented exceptions include approval owner and expiration.
7. Operational readiness validation: runbook walkthrough completed for IP update, access revocation, and incident triage.

**Decisions**
- Chosen by user: Standard SKU.
- Chosen by user: Public endpoint remains enabled but restricted to selected IP ranges.
- Chosen by user: Entra ID-only authentication model (disable local/SAS auth).
- Chosen by user: Include full baseline-aligned governance (Policy + monitoring + operational controls).
- Chosen by user: Service Bus allow-list source is AKS private-cluster egress via Standard Load Balancer using a single static public IP.
- Included scope: architecture pattern, control mapping, governance model, and verification approach for a single namespace + one-to-many queues.
- Excluded scope: Premium-only capabilities (private endpoints, CMK) except as migration triggers.

**Further Considerations**
1. Topology locked: one namespace with one-to-many queues; add new queues by bounded context, throughput need, and ownership boundary.
2. Cross-region resilience recommendation: Option A active/passive DR with documented failover. Option B geo-distributed app failover with duplicate messaging paths.
3. Queue growth recommendation: Option A keep a lean queue set with strict contract/versioning and retention standards. Option B add queues only for bounded contexts, each with separate RBAC and monitoring.
4. Change management recommendation: Option A manual firewall updates with CAB approval. Option B automated allow-list pipeline with policy checks and mandatory approvals.