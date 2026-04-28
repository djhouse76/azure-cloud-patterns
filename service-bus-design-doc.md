# Public Secure Azure Service Bus Baseline Pattern for Standard Tier

## Summary

This design defines a secure-by-default reference pattern for Azure Service Bus Standard tier where the public endpoint remains enabled but is restricted to approved source IP ranges. The baseline uses Microsoft Entra ID-only authentication, queue-scoped least-privilege RBAC, minimum TLS 1.2, centralized diagnostics, alerting, Azure Policy guardrails, and operator governance aligned to the Azure Service Bus security baseline.

The scope is one Service Bus namespace hosting one-to-many queues. Approved ingress originates from a private AKS cluster through a Standard Load Balancer with a single static outbound public IP. The design prioritizes controlled public exposure, auditable change management, and clear migration triggers for Premium when private connectivity, customer-managed keys, or stronger isolation become mandatory.

## Context and Problem

Azure Service Bus must be exposed over a public endpoint because the selected baseline targets Standard tier and a public-but-restricted connectivity model. Public reachability increases attack surface unless tightly governed through network controls, identity-only authentication, authorization scoping, monitoring, and policy enforcement.

The required pattern must satisfy these constraints:

- Service Bus SKU is Standard.
- The namespace public endpoint remains enabled.
- Access is restricted to selected IPv4 ranges only.
- Producer and consumer workloads run on AKS and must present a stable outbound IP to avoid firewall drift.
- SAS and local authentication must be disabled; Microsoft Entra ID is the only authentication model.
- Authorization must follow least privilege, with queue-scoped sender and receiver rights for workloads and tightly controlled administrative access for operators.
- The solution must be operationally governable and continuously auditable.

### Assumptions

- All producer and consumer workloads can use Azure SDKs that support Microsoft Entra ID authentication over TLS 1.2 or later.
- AKS uses Standard Load Balancer outbound configuration and can be pinned to a user-managed public IP.
- The Service Bus namespace, AKS cluster, Log Analytics workspace, and policy assignments are deployed in approved Azure regions.
- Workload identities and operator identities exist in the same Microsoft Entra tenant as the Service Bus resource.
- Sentinel integration is optional and only applies where a SIEM deployment already exists.
- The initial deployment is single-region. Cross-region resilience is discussed as a future-state alternative, not part of the baseline build.
- Message schema, queue TTL, duplicate detection, sessions, and application retry policies are owned by application teams and are outside this infrastructure baseline unless explicitly required later.

## Goals

- Provide a Standard-tier Service Bus reference architecture with controlled public connectivity.
- Restrict namespace access to approved AKS egress IP ranges with deny-by-default network rules.
- Enforce Microsoft Entra ID-only authentication by disabling local and SAS authentication.
- Apply least-privilege RBAC at queue scope for producer and consumer workloads.
- Establish centralized observability with diagnostics, metrics, alerting, and compliance reporting.
- Enforce baseline controls through Azure Policy and operational governance.
- Document clear operational runbooks, review cadences, and Premium migration triggers.

## Non-Goals

- Private connectivity patterns such as Private Endpoint.
- Customer-managed keys or double encryption.
- Multi-region active-active or geo-disaster-recovery implementation.
- Application-specific message contract design.
- End-to-end AKS cluster design beyond stable outbound IP requirements.
- Premium-only Service Bus capabilities unless discussed as migration triggers.

## Requirements

### Functional Requirements

- Deploy one Azure Service Bus namespace on Standard tier.
- Support one-to-many queues within the namespace.
- Enable the public endpoint and restrict access to selected IPv4 CIDR ranges only.
- Configure namespace network default action as deny.
- Allow only approved AKS outbound public IP ranges; trusted Microsoft services are exception-only.
- Disable local and SAS authentication on the namespace.
- Use managed identities for all producer and consumer workloads.
- Assign Azure Service Bus Data Sender and Azure Service Bus Data Receiver roles at queue scope per workload.
- Reserve broader administrative access for platform operators through separated roles and just-in-time elevation.
- Enforce minimum TLS version 1.2.
- Send Service Bus logs and metrics to Log Analytics.
- Generate alerts for failed authentication, firewall denials, unusual messaging behavior, and configuration changes.
- Enforce posture through Azure Policy, including local auth disabled, minimum TLS, diagnostics enabled, approved SKU and regions, and tagging.
- Track compliance status through Defender for Cloud where enabled.
- Provide runbooks for IP allow-list changes, access revocation, incident response, and periodic reviews.

### Non-Functional Requirements

- Default-deny network posture for public access.
- Stable and auditable AKS egress identity to prevent unplanned firewall changes.
- Least-privilege authorization with clear ownership boundaries.
- Centralized evidence for audit, incident investigation, and compliance reporting.
- Infrastructure changes must be controlled through CAB approval or an approved deployment pipeline.
- Exceptions must be documented with owner, rationale, approval, and expiration.
- The baseline must be repeatable across environments with consistent controls.

## Proposed Design

### High-Level Architecture

The design uses a single Azure Service Bus Standard namespace hosting multiple queues. Producer and consumer workloads run on a private AKS cluster. AKS egress is forced through a Standard Load Balancer configured with one user-managed static public IP. That public IP is the primary source added to the Service Bus network allow-list.

Traffic flow is:

1. A workload in AKS obtains a Microsoft Entra token using its managed identity.
2. The workload connects over TLS 1.2+ to the Service Bus public endpoint.
3. Service Bus evaluates the source IP against namespace network rules.
4. If the source IP is allowed, Service Bus validates the Entra token.
5. RBAC at queue scope determines whether the identity can send or receive.

Management and governance flow is:

1. Infrastructure is provisioned and updated through approved change control.
2. Azure Policy validates or enforces baseline settings.
3. Diagnostic logs and metrics are sent to Log Analytics and optionally forwarded to Sentinel.
4. Alerts notify operations on failed authentication, denied network access, and configuration changes.
5. Defender for Cloud tracks compliance posture and remediation tasks.

### Trust Boundaries

- AKS private network boundary: internal workloads are not directly trusted by Service Bus; they must egress through the approved public IP and authenticate with Entra ID.
- Public internet boundary: the Service Bus endpoint is internet-addressable, but access is constrained by IP allow-list and identity checks.
- Azure control plane boundary: operators administer Service Bus through Azure RBAC, PIM, policy, and audited change workflows.
- Logging and compliance boundary: Log Analytics and policy systems are the authoritative audit trail for baseline evidence.

### Phase 1: Network Security Foundation

- Configure Service Bus networking with public network access enabled and selected networks only.
- Set default network action to deny.
- Add explicit allow-list entries for approved IPv4 CIDRs. The initial expected source is the AKS outbound static public IP.
- Keep trusted Microsoft services disabled by default. Enable only through documented exception approval when a dependent Azure service requires it and cannot use the current allow-list model.
- Configure AKS Standard Load Balancer outbound settings to use a user-managed static public IP.
- Validate the effective outbound IP from AKS after deployment.
- Treat outbound IP changes as controlled changes requiring CAB approval or an approved pipeline, because any drift will immediately affect Service Bus connectivity.

### Phase 2: Identity and Authorization

- Disable local and SAS authentication at the namespace level so all client access requires Microsoft Entra ID.
- Require managed identities for all workload access. No connection strings are distributed to applications.
- Assign Azure Service Bus Data Sender only to producer identities and only at the required queue scope.
- Assign Azure Service Bus Data Receiver only to consumer identities and only at the required queue scope.
- Avoid namespace-scope data roles for applications.
- Reserve namespace-level administrative permissions for platform operators only.
- Use separate administrative roles for resource management versus security/compliance oversight where practical.
- Require PIM or equivalent just-in-time activation for privileged roles.
- Define a break-glass path with documented approval, time bounds, and post-use audit review.

### Phase 3: Data Protection and Transport

- Set minimum TLS version to 1.2 on the namespace.
- Validate client SDK and runtime support for TLS 1.2+ before production rollout.
- Use the platform default encryption at rest provided by Azure Service Bus Standard.
- Document customer-managed keys as unsupported in this baseline and a migration trigger to Premium if mandated.

### Phase 4: Monitoring, Detection, and Compliance

- Enable diagnostic settings for Service Bus resource logs and metrics to a central Log Analytics workspace.
- Where Sentinel is already deployed, forward high-value signals into SIEM analytics.
- Configure alerts for:
  - Authentication failures, including attempts to use disabled local/SAS auth.
  - Network-rule denials from non-approved IP addresses.
  - Unusual sender or receiver patterns such as sudden volume changes or abnormal error rates.
  - Administrative changes to namespace network configuration, authentication settings, TLS settings, or diagnostic settings.
- Assign Azure Policy initiatives and definitions to enforce or audit:
  - Local authentication disabled.
  - Minimum TLS version configured.
  - Diagnostic settings enabled.
  - Approved SKU and region usage.
  - Mandatory resource tags.
- Enable Defender for Cloud recommendations and compliance tracking for the resource group or subscription scope where this pattern is deployed.
- Define remediation workflow for noncompliant controls, including ticket ownership and target SLA.

### Phase 5: Operational Runbook and Artifacts

- Produce a logical architecture diagram showing AKS egress, Service Bus, Log Analytics, policy, and operator control paths.
- Produce a baseline control mapping matrix to the Azure Service Bus security baseline.
- Produce an RBAC matrix showing identities, scopes, and approved actions.
- Produce runbooks for:
  - IP allow-list changes.
  - Access revocation.
  - Incident triage for failed auth or blocked network access.
  - Quarterly access reviews and monthly firewall reviews.
- Define lifecycle control cadence:
  - Quarterly role recertification.
  - Monthly firewall allow-list review.
  - Ongoing policy drift review through compliance reports.
- Record trade-offs and migration triggers for Premium.

### Implementation Approach

The recommended implementation is infrastructure-as-code plus policy-as-code with a controlled release process.

- Use Bicep, ARM, or Terraform to declare the namespace, queue set, network rules, diagnostic settings, alerts, and RBAC assignments.
- Use Azure Policy assignments at management group, subscription, or landing-zone scope based on the existing governance model.
- Manage IP allow-list updates through a change-controlled parameter source rather than manual portal edits wherever possible.
- Require deployment approvals for any changes affecting:
  - AKS outbound public IP.
  - Service Bus network allow-list.
  - Authentication mode.
  - TLS minimum version.
  - Privileged role assignments.

## Alternatives Considered

### Alternative A: Service Bus Premium with Private Endpoint

This would reduce public exposure by moving to private connectivity and would also allow customer-managed keys and stronger isolation. It is not selected because the current decision is Standard tier and a public-but-restricted baseline. Premium remains the preferred future option if regulatory requirements, isolation needs, or key-management requirements increase.

Trade-offs:

- Better network isolation and stronger security posture.
- Higher cost and potentially more operational complexity.
- Solves public endpoint exposure concerns more cleanly than IP restrictions.

### Alternative B: Public Endpoint with SAS Authentication Enabled

This would simplify some client integrations but is rejected because it weakens credential governance, increases secret handling risk, and conflicts with the requirement for Entra ID-only access. It also reduces auditability compared to managed identity and RBAC.

Trade-offs:

- Easier integration for legacy clients.
- Higher secret sprawl, rotation burden, and misuse risk.
- Does not meet the baseline requirement.

### Alternative C: Multiple Namespaces for Queue Isolation

This would provide stronger resource-level separation across domains, environments, or ownership boundaries. It is not selected for the baseline because the defined scope is one namespace with one-to-many queues. Multiple namespaces may become appropriate if queue ownership, throughput isolation, or blast-radius reduction outweigh operational simplicity.

Trade-offs:

- Better tenancy and isolation.
- Higher management overhead and cost.
- More policy, diagnostics, and RBAC surfaces to maintain.

## Data Model and Interfaces

### Resource Model

| Resource | Scope | Purpose | Key Configuration |
|---|---|---|---|
| Service Bus namespace | Per environment | Messaging control plane and queue host | Standard SKU, public access enabled, selected networks only, local auth disabled, TLS 1.2 minimum |
| Queues | Within namespace | Workload messaging nodes | One-to-many topology, ownership mapped per bounded context |
| AKS outbound public IP | Per cluster/environment | Stable egress identity | User-managed static public IP attached to Standard Load Balancer outbound config |
| Managed identities | Per workload | Client authentication to Service Bus | User-assigned or system-assigned based on platform standard |
| RBAC assignments | Queue or namespace scope | Least-privilege authorization | Sender/Receiver at queue scope; admin roles for operators |
| Diagnostic settings | Namespace scope | Log and metric export | Log Analytics target; optional Sentinel forwarding |
| Azure Policy assignments | Subscription or resource group scope | Prevent and detect drift | Deny, audit, deployIfNotExists as appropriate |
| Alerts | Subscription/resource scope | Security and operations detection | Auth failures, denials, anomalies, config changes |

### Configuration Data Elements

| Element | Type | Owner | Notes |
|---|---|---|---|
| Approved source CIDRs | List of IPv4 CIDRs | Platform/network team | Must include AKS outbound public IP; changes are controlled |
| Queue ownership map | Queue-to-team mapping | Platform plus app owners | Used for RBAC and review cadence |
| Workload identity mapping | Workload-to-managed-identity mapping | Platform/app owners | Each workload identity mapped to sender or receiver rights |
| Required tags | Key/value policy set | Governance team | Applied consistently across environments |
| Exception register | Structured record | Governance/SOC | Includes owner, approval, reason, expiry, and review date |

### Interfaces

#### Client-to-Service Bus Interface

- Protocol: AMQP over TLS or HTTPS over TLS, subject to supported Azure Service Bus SDK behavior.
- Authentication: Microsoft Entra token only.
- Authorization: Azure RBAC evaluated at queue scope.
- Network gate: Service Bus namespace IP/network rules evaluated before authorized operations succeed.

#### Management Interface

- Azure Resource Manager and Azure Policy are the control plane for namespace configuration, diagnostics, and compliance.
- Azure RBAC controls operator access to manage resources.
- PIM controls temporary elevation where available.

#### Observability Interface

- Service Bus diagnostics and metrics are exported to Log Analytics.
- Alert rules query logs and metrics and notify operations tooling.
- Sentinel, if present, consumes selected signals for correlation and incident workflows.

### RBAC Matrix

| Identity Type | Scope | Role | Allowed Actions |
|---|---|---|---|
| Producer workload managed identity | Queue | Azure Service Bus Data Sender | Send to assigned queue only |
| Consumer workload managed identity | Queue | Azure Service Bus Data Receiver | Receive from assigned queue only |
| Platform operator | Namespace/resource group | Administrative role per platform standard | Manage namespace settings, queues, diagnostics, and network rules |
| Security/compliance operator | Subscription/resource group | Reader or security oversight roles per platform standard | Review logs, policy compliance, alerts, and recommendations |
| Break-glass operator | Time-bound elevated scope | Approved emergency role | Emergency recovery only with post-incident audit |

## Security and Compliance

### Security Controls

- Public endpoint exposure is limited through selected networks and explicit CIDR allow-list rules.
- Default network action is deny.
- Trusted Microsoft services are disabled by default and require documented exception approval.
- Local and SAS authentication are disabled to eliminate shared-secret access paths.
- Managed identities remove application secret distribution and reduce credential leakage risk.
- Queue-scoped RBAC reduces blast radius compared to namespace-wide application permissions.
- Minimum TLS 1.2 enforces current transport security requirements.
- Platform-managed encryption at rest is enabled by the service.
- Administrative access is controlled through role separation, least privilege, and just-in-time elevation.

### Compliance Alignment

This baseline is intended to map to Azure Service Bus security baseline controls in these areas:

- Network access restriction and segmentation.
- Strong identity and authentication controls.
- Least-privilege authorization.
- Logging, monitoring, and alerting.
- Secure configuration enforcement.
- Continuous compliance assessment and remediation tracking.

### Exception Handling

Any deviation from the baseline requires a documented exception with:

- Business and technical justification.
- Approval owner.
- Time-bound expiration.
- Compensating controls.
- Review date and retirement plan.

## Performance and Scalability

- Standard tier is selected for baseline cost and simplicity; it is suitable where workload scale and isolation needs fit Standard service limits.
- A single namespace with multiple queues simplifies operations but concentrates throughput and quota consumption into one namespace boundary.
- Queue growth must be controlled by bounded context and ownership to avoid uncontrolled namespace sprawl.
- Monitoring should track queue depth, throttling indicators, request rates, and error rates to identify when the namespace is approaching operational limits.
- The static AKS outbound IP introduces a single approved egress path, which supports security governance but must be treated as critical connectivity infrastructure.
- Cross-region resilience is not implemented in this baseline; if recovery objectives exceed what a single-region deployment can support, a future-state design is required.

### Premium Migration Triggers

Move to Premium when one or more of the following apply:

- Private Endpoint or private-only connectivity becomes mandatory.
- Customer-managed keys are required.
- Regulatory obligations prohibit public endpoint usage, even with IP restrictions.
- Throughput isolation or predictable performance requires dedicated messaging resources.
- Namespace-level multi-tenant contention becomes operationally unacceptable.

## Operational Considerations

### Day-2 Operations

- Treat the AKS outbound public IP as a controlled dependency. Any change must be coordinated with a namespace allow-list update.
- Prefer automated deployment workflows with approvals over manual portal configuration to reduce drift.
- Review queue ownership regularly to ensure RBAC remains aligned with current application boundaries.
- Ensure disabled local auth remains continuously enforced; do not rely on one-time deployment settings only.

### Runbooks

#### IP Allow-List Change Runbook

- Confirm business need and expected source IP or CIDR.
- Validate whether the change is temporary or permanent.
- Obtain CAB or pipeline approval.
- Update infrastructure definition and deploy.
- Validate connectivity from the approved source and denial from a non-approved source.
- Record change evidence and update the exception register if applicable.

#### Access Revocation Runbook

- Identify identity and current role assignments.
- Remove queue-scoped or admin RBAC assignments.
- Validate access failure after revocation.
- Review recent activity for misuse indicators.
- Document completion and ticket closure evidence.

#### Incident Triage Runbook

- Determine whether the issue is network denial, authentication failure, or authorization mismatch.
- Review Service Bus diagnostics, Activity Log, and policy compliance state.
- Confirm whether recent changes affected outbound IP, RBAC, auth mode, or diagnostics.
- Escalate to platform, security, or application owners based on failure domain.
- Capture timeline, remediation, and residual risk.

### Review Cadence

- Monthly firewall allow-list review.
- Quarterly RBAC recertification.
- Ongoing policy compliance review with remediation SLAs.
- Periodic validation of alert fidelity using synthetic test events.

## Testing Strategy

### Pre-Production Validation

- Connectivity validation:
  - Verify the AKS static outbound IP can connect.
  - Verify non-approved public IPs are denied.
- Authentication validation:
  - Confirm SAS/local auth attempts fail.
  - Confirm Entra token-based access using managed identity succeeds.
- Authorization validation:
  - Confirm sender identities cannot receive.
  - Confirm receiver identities cannot send.
  - Confirm queue-scoped permissions do not grant access to other queues.
- Security configuration validation:
  - Confirm TLS minimum is 1.2 or higher.
  - Confirm local auth is disabled.
  - Confirm selected networks mode is active.
  - Confirm explicit allow-list entries are present.
- Observability validation:
  - Confirm logs and metrics arrive in Log Analytics.
  - Trigger synthetic failed auth and blocked network events and verify alerting.
- Compliance validation:
  - Confirm Azure Policy reports green for required controls.
  - Confirm exceptions include owner and expiration.
- Operational readiness validation:
  - Execute runbook walkthroughs for IP updates, access revocation, and incident triage.

### Ongoing Validation

- Schedule recurring synthetic tests for blocked IP and failed auth scenarios.
- Review alert noise and tune thresholds without reducing required detection coverage.
- Audit random queue-level RBAC assignments each quarter for least-privilege compliance.

## Migration and Rollout Plan

### Rollout Sequence

1. Approve target region, naming, tags, and ownership model.
2. Reserve and deploy the user-managed static public IP for AKS outbound traffic.
3. Configure AKS to use the static outbound IP and verify effective egress.
4. Deploy the Service Bus namespace with Standard SKU, selected networks mode, deny-by-default rules, TLS 1.2 minimum, and local auth disabled.
5. Create required queues according to approved bounded contexts and ownership.
6. Create or bind managed identities for producer and consumer workloads.
7. Assign queue-scoped RBAC roles for sender and receiver identities.
8. Enable diagnostics to Log Analytics and configure alert rules.
9. Assign Azure Policy controls and validate compliance state.
10. Enable Defender for Cloud tracking and define remediation ownership.
11. Execute verification steps and capture evidence.
12. Transition to operational ownership with approved runbooks and review schedule.

### Migration Notes

- Existing clients using connection strings or SAS must be updated to use Entra ID before cutover.
- Any workload with dynamic or unknown outbound IP behavior must be stabilized before onboarding.
- Existing broad namespace-level application permissions should be decomposed into queue-scoped roles during migration.

### Rollback Considerations

- Roll back configuration changes through infrastructure deployment history rather than ad hoc portal edits.
- Do not temporarily re-enable local auth as a convenience rollback path; treat that as a separate exception requiring explicit approval.
- If AKS outbound IP changes unexpectedly, restore the prior egress path or update the allow-list immediately under incident control.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| AKS outbound IP changes unexpectedly | Service Bus connectivity outage | Use a user-managed static public IP, change control, and post-change validation |
| Public endpoint remains exposed to the internet | Increased attack surface | Deny-by-default network rules, narrow allow-list, Entra-only auth, logging, and alerting |
| Legacy clients cannot use Entra ID | Migration delays or temporary incompatibility | Validate SDK support early and track remediation before production cutover |
| Overly broad RBAC assignments | Excess privilege and larger blast radius | Enforce queue-scoped roles, quarterly recertification, and policy or review controls |
| Diagnostics disabled or misconfigured | Loss of audit and incident evidence | Enforce via policy, alert on diagnostic setting changes, and validate during rollout |
| Trusted services exception is overused | Weakens network restriction intent | Require exception approval, expiration, and compensating controls |
| Single namespace becomes a contention point | Throughput or operational bottlenecks | Monitor utilization, govern queue growth, and define Premium or multi-namespace triggers |
| Single-region deployment cannot meet recovery needs | Extended outage during regional incidents | Document as a future-state gap and escalate if business continuity requirements change |

## Open Questions

- Which exact Azure Policy definitions or initiative set will be used in the target landing zone for Service Bus controls?
- Which operator roles in the tenant will map to namespace administration, security oversight, and break-glass access?
- Will managed identities be system-assigned, user-assigned, or mixed based on platform standards?
- What thresholds define unusual sender/receiver patterns for alerting in each environment?
- Is Sentinel available in the target environment, or will Log Analytics-only monitoring be used initially?
- What queue naming and ownership convention will be adopted to enforce bounded context growth?
- What is the approved process model for changes: manual CAB, deployment pipeline approvals, or both?

## Decision Log

| Decision | Status | Rationale | Consequence |
|---|---|---|---|
| Use Azure Service Bus Standard SKU | Approved | Meets current scope and cost target | No Private Endpoint or CMK in baseline; future Premium migration may be required |
| Keep public endpoint enabled but restricted to selected IP ranges | Approved | Standard-tier public-but-restricted baseline requirement | Public exposure remains and must be governed through firewall rules and monitoring |
| Use one namespace with one-to-many queues | Approved | Simplifies baseline architecture and aligns to scope | Requires queue ownership discipline and monitoring for scale limits |
| Use AKS private-cluster egress through a single static public IP | Approved | Provides stable source identity for Service Bus allow-listing | Outbound IP becomes a critical dependency requiring controlled change |
| Disable local and SAS authentication | Approved | Enforces Entra ID-only access and removes shared-secret paths | Legacy connection-string clients must be remediated |
| Use managed identities with queue-scoped sender/receiver roles | Approved | Implements least privilege and auditable workload identity | RBAC administration requires ownership mapping and review discipline |
| Enforce TLS 1.2 minimum | Approved | Meets transport security baseline | Older clients must be validated or upgraded |
| Centralize diagnostics in Log Analytics with policy-backed enforcement | Approved | Supports monitoring, audit, and compliance evidence | Additional operational overhead for alert tuning and retention management |
| Treat trusted Microsoft services as exception-only | Approved | Preserves the intent of restricted public exposure | Exception governance process must be maintained |
| Define Premium migration triggers for private endpoint, CMK, or regulatory needs | Approved | Keeps the baseline pragmatic while documenting future-state conditions | Migration path must be revisited if requirements tighten |