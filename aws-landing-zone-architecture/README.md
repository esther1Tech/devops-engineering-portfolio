# AWS Landing Zone Architecture

## Purpose and Scope

An AWS landing zone is the secure, governed foundation on which application environments are built. It establishes account boundaries, identity controls, network connectivity, shared security services, and operational guardrails before application workloads are deployed.

This document describes a reference design for a four-environment platform: **DEV, QA, UAT, and PROD**. It covers the requested layers, personas, network fabric, capacity-planning method, baseline services, data isolation, and governance.

> **Blueprint-dependent values:** Exact VPC, subnet, NAT Gateway, Transit Gateway, and endpoint counts depend on the approved environment design and remain **TBD** here. Any numerical example below is explicitly illustrative, not a claim about a deployed environment. Confirm the AWS Region, Availability Zones (AZs), CIDR allocations, account topology, endpoint list, and scale targets before implementation.

## 1. Architecture Layers

The layers separate responsibilities and make ownership, access, and change impact easier to reason about. They are logical boundaries; the exact AWS account and VPC placement depends on the approved landing-zone design.

| Layer | Main responsibility | Typical components |
|---|---|---|
| **CI/CD & DevTools** | Stores source and artifacts, validates Terraform, builds images, and coordinates approved deployments. | Git hosting, CI runners, Terraform, artifact and container registries |
| **Edge** | Receives and filters external traffic before it reaches application services. | DNS, WAF, public load balancers, ingress |
| **Network** | Connects network zones and controls allowed paths between them. | VPCs, subnets, route tables, Transit Gateway, firewalls, VPC endpoints |
| **Infra** | Provides shared compute and foundational platform capabilities. | EKS foundation, node infrastructure, shared services, KMS keys |
| **App** | Runs product APIs, workers, and other application workloads. | EKS workloads, services, autoscaling, workload identities |
| **Data** | Hosts durable and ephemeral data systems behind restricted network paths. | MySQL, Valkey, Neo4j, MongoDB, backups |
| **Identity & Access** | Authenticates people and workloads and authorizes their actions. | IAM Identity Center, IAM roles, federation, Vault, policy controls |
| **Observability** | Collects operational and security evidence and supports detection and response. | Logs, metrics, traces, CloudTrail, GuardDuty, AWS Config |

### Why isolate layers?

- **Limit blast radius:** A compromised or misconfigured edge component should not automatically gain a route to databases or administrative systems.
- **Apply least privilege:** Each team, workload, and automation role receives only the permissions and network paths it needs.
- **Change components independently:** Modular Terraform lets teams plan and review a bounded module change. Separate states and deployment stages can reduce unrelated change coupling.
- **Improve auditability:** Ownership, approval requirements, and logs can be associated with a layer and environment.
- **Support availability:** A well-designed module boundary helps avoid changing unrelated infrastructure, but modular Terraform alone does **not** guarantee zero downtime. Availability still depends on resource lifecycle, compatibility, redundancy, rollout strategy, and testing.

Modules should expose deliberate inputs and outputs, pin compatible versions, and avoid circular dependencies. Shared foundations should be changed through reviewed plans and staged rollouts, not ad hoc global applies.

## 2. Environments, Personas, and Access Boundaries

### Environment isolation

Use separate AWS accounts for DEV, QA, UAT, and PROD wherever possible, grouped under an AWS Organization. This makes identity policies, quotas, logs, network controls, and failures easier to isolate. Central security and audit accounts can receive logs without sharing application data stores.

| Environment | Intended use | Change control |
|---|---|---|
| **DEV** | Development and integration testing with synthetic or appropriately masked data. | Frequent changes; developers may use approved self-service workflows. |
| **QA** | Repeatable functional, integration, and security verification. | Changes flow through CI with test evidence; avoid persistent manual drift. |
| **UAT** | Business acceptance against a release candidate and production-like configuration. | Restricted changes and explicit release approval. |
| **PROD** | Live customer-facing workloads and regulated data. | Protected deployments, least privilege, separation of duties, and audited emergency access. |

### Persona access matrix

Access should be granted through federated identities and short-lived role sessions, not shared IAM users or long-lived access keys. This matrix is a policy model; the actual IAM policies and permission sets must be reviewed against job responsibilities.

| Persona | DEV | QA | UAT | PROD |
|---|---|---|---|---|
| **Cloud Admin / Ops** | Administer approved foundations and troubleshoot. Elevated actions are logged. | Restricted operational role; elevated changes reviewed. | Limited operational role; changes approved and audited. | No standing broad admin. Use tightly scoped operations roles; exceptional access follows break-glass procedure. |
| **Application Developer** | Read application infrastructure and use approved development workflows. Write changes through code and CI; direct console changes should be limited and temporary. | Primarily read/diagnose; request changes through pull requests and pipeline. | Read-only observation and UAT support; no direct infrastructure writes. | Read-only access to approved operational views; no direct code or infrastructure changes. |
| **CI/CD Pipeline** | Deploy approved builds and infrastructure plans using DEV-scoped roles. | Deploy only after required checks pass, using QA-scoped roles. | Deploy a release candidate after approval, using UAT-scoped roles. | Deploy only the approved artifact/plan after production gates, using a PROD-scoped role with narrowly defined permissions. |

As changes move toward production, the ability to make direct changes contracts. Developers can still propose and review changes, but production changes are made by an auditable pipeline after code review, automated checks, and required approvals. This reduces accidental changes, unreviewed privilege escalation, and uncertainty about which code is running. Read-only access must still be sufficient for legitimate troubleshooting, dashboards, and logs, with sensitive data appropriately restricted.

### Terraform workspaces, backends, and variable files

Terraform workspaces select separate state instances for one configuration. They are **not** security boundaries: workspaces do not create AWS accounts, isolate IAM permissions, or prevent an operator from selecting the wrong workspace. For production isolation, use separate accounts, narrowly scoped roles, and distinct backend state locations. Separate root configurations per environment are often clearer than using one workspace for every environment.

`.tfvars` files provide environment-specific input values; they do not configure or secure the backend by themselves. A typical layout is:

```text
infrastructure/
├── modules/
│   ├── network/
│   └── eks/
└── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── backend.hcl
│   │   └── dev.tfvars
│   ├── qa/
│   │   ├── main.tf
│   │   ├── backend.hcl
│   │   └── qa.tfvars
│   ├── uat/
│   │   ├── main.tf
│   │   ├── backend.hcl
│   │   └── uat.tfvars
│   └── prod/
│       ├── main.tf
│       ├── backend.hcl
│       └── prod.tfvars
```

Use one reviewed environment root and state configuration per environment, with reusable versioned modules. Backend configuration should identify the correct state location and locking behavior. Do not put backend credentials or secret values in `.tfvars`, `backend.hcl`, or source control. Authenticate through CI federation or a secure credential provider and assume a role scoped to the target account.

Use explicit environment selection and separate plan/apply stages. CI should initialize the environment root with its backend configuration, assume the scoped account role, run `fmt`, `validate`, tests and policy checks, and pass only that environment's `.tfvars` file to `plan`. Publish the saved plan for access-controlled review; a production `apply` must consume that same approved plan after the designated approval. If the plan becomes stale, generate and review a new one. Protect state and plan files because they can contain sensitive values. Never let an untrusted pull request receive production credentials or choose its own deployment target.

### Emergency break-glass access

Break-glass access is for a time-critical incident when normal deployment paths are unavailable, not a shortcut around review:

1. Open or reference an incident/service-desk ticket with impact, reason, and requested scope.
2. Obtain approval from the designated incident commander or independent approver, following the organization's emergency policy.
3. Use a dedicated, narrowly scoped, time-limited role through the approved identity system; require strong authentication and record the session.
4. Make the smallest necessary change, capture commands and evidence, and avoid untracked console edits where a reviewed Terraform change can be applied safely.
5. Revoke or allow the temporary grant to expire, verify the service, and reconcile any emergency change back into version-controlled Terraform.
6. Review the event afterward and document follow-up actions.

### Runtime secrets: Vault and Kubernetes configuration

ConfigMaps are for non-sensitive configuration and should never hold credentials. Kubernetes Secrets are intended for sensitive values, but their security depends on cluster encryption at rest, strict RBAC, audit controls, delivery practices, and rotation; base64 encoding is not encryption.

HashiCorp Vault can centralize policy-controlled secret access, issue short-lived or dynamic credentials where supported, rotate secrets, and provide access audit records. Workloads should authenticate using a workload identity and retrieve only their authorized secrets. Avoid baking secrets into images, Terraform state, logs, or plaintext manifests. Vault complements rather than replaces AWS IAM, KMS, and Kubernetes access controls.

## 3. Network and Interconnect Fabric

The requested **Edge, App, Data, and Ops** zones are trust boundaries. They may be separate VPCs or separately routed subnets; the exact placement must follow the approved network blueprint.

```text
Internet
   |
   v
Edge zone (public ingress, WAF / load balancer)
   |
   | Explicitly allowed application routes
   v
App zone (private EKS nodes and services)
   |
   | Only required database and service ports
   v
Data zone (private database endpoints)

Ops zone (restricted management, monitoring, and shared tooling)
   |
   +---- Controlled paths through Transit Gateway / inspection
```

### Transit Gateway and route isolation

AWS Transit Gateway is a regional network hub for connecting VPCs and on-premises networks. It simplifies connectivity, but it is not a firewall by itself. Use separate Transit Gateway route tables and explicit VPC attachment associations/propagations to control which networks learn routes to which other networks. Add VPC route-table entries only for approved paths, then enforce service-level access with security groups and, where required, network firewalls.

For example, the Edge attachment should have routes only to approved App ingress subnet CIDRs. It should not receive a route to the Data VPC. App workloads may reach specific database subnets and ports; the Data zone should not accept arbitrary connections from Edge. Review both forward and return routing, because asymmetric or overly broad propagated routes can defeat the intended isolation. Deny-by-default security groups and flow-log review provide additional safeguards.

### VPC endpoints and private service access

VPC endpoints provide private connectivity to supported AWS services without sending that service traffic through a public internet gateway. **Interface endpoints** use AWS PrivateLink and create endpoint network interfaces in selected subnets. **S3 gateway endpoints** are a different endpoint type: they add private routes to S3 and are not PrivateLink interface endpoints.

For private EKS workers to pull images from private ECR without an internet default route, the VPC generally needs:

- ECR API interface endpoint (`ecr.api`).
- ECR Docker registry interface endpoint (`ecr.dkr`).
- S3 gateway endpoint for the image layers stored in S3.
- Private DNS and endpoint policies that allow only required registries and buckets.
- Security group rules allowing worker nodes to reach the interface endpoints over HTTPS.

Add endpoints for other required AWS services (for example, CloudWatch Logs, STS, or KMS) only after confirming workload and platform needs. Endpoint costs, regional availability, DNS, policies, and route tables must be included in the design. If workloads need external destinations not covered by endpoints, provide controlled egress through approved NAT/firewall/proxy paths or other private connectivity. Do not add a blanket `0.0.0.0/0` internet route merely to make image pulls work.

## 4. Network Capacity Planning

### Exact deployment counts

The design identifies four environments and four network zones, but does not define whether each zone is a VPC, which accounts share network services, how many AZs are required, or which endpoints are deployed in which VPC. Therefore, deployed totals cannot be calculated until those inputs are defined.

| Resource | Count status | Inputs needed |
|---|---|---|
| Total VPCs | **TBD** | Whether zones are separate VPCs, shared-services design, and environment/account topology |
| Public and private subnets | **TBD** | VPC placement, AZ count, workload tiers, and CIDRs |
| Internet Gateways | **TBD** | Number of VPCs that require direct public ingress/egress |
| NAT Gateways | **TBD** | Egress design, AZ resilience target, and which private subnets need public destinations |
| Transit Gateway(s) and attachments | **TBD** | Number of network hubs, connected VPCs/accounts, and routing domains |
| VPC endpoints | **TBD** | Required AWS services, endpoint types, VPC placement, and selected AZs |

Do not treat “four environments” multiplied by “four tiers” as a confirmed VPC count. That multiplication is valid only if the approved design explicitly assigns one VPC to every tier in every environment.

### Sizing method and parameter model

1. **Choose AWS Regions and AZs.** Decide the required failure tolerance and confirm that the selected services and instance types are available there.
2. **Allocate non-overlapping CIDRs.** Reserve address space for every environment, VPC, and future expansion. Check for overlap with corporate, partner, and on-premises networks.
3. **Estimate addresses per subnet.** Include worker node ENIs, Pod IP allocation (including CNI behavior and maximum scale), load balancers, interface endpoint ENIs, and growth headroom. AWS reserves five IPv4 addresses in each subnet. For an IPv4 subnet of prefix `/p`, usable addresses are `2^(32-p) - 5` before reserving addresses for expected consumers.
4. **Create subnets per AZ and trust tier.** Place public ingress subnets only where public-facing resources require them. Keep application, data, and operations workloads in private subnets unless a documented design requires otherwise.
5. **Select egress and resilience.** A NAT Gateway per AZ that needs egress can reduce cross-AZ dependency and provide AZ-level resilience, but costs more. A centralized egress design is also possible and must be routed and inspected deliberately.
6. **Enumerate endpoints.** For each VPC, list required interface endpoint services and selected AZs; each interface endpoint needs an ENI/address in each selected AZ. S3 gateway endpoints attach to route tables and do not consume interface endpoint ENIs.
7. **Validate and reserve capacity.** Check CIDR capacity, EKS Pod/IP allocation, endpoint addresses, route limits, quota limits, failover behavior, and future growth before applying.

Terraform input parameters should make these decisions explicit, for example:

```text
region
availability_zones
environment
vpc_cidrs
subnet_cidrs_by_zone_and_tier
enable_public_ingress
nat_gateway_mode
transit_gateway_id
transit_gateway_route_table_associations
interface_endpoint_services
interface_endpoint_azs
s3_gateway_endpoint_route_tables
```

This is a parameter checklist, not a ready-to-apply module interface. The exact variable names and values must match the Terraform modules and approved IP plan.

### Illustrative calculation only

If—and only if—the design selected four separate VPCs per environment, three AZs, one public ingress subnet per AZ in each Edge VPC, and one private subnet per AZ in each App, Data, and Ops VPC, then:

- VPCs: `4 environments × 4 VPCs = 16`.
- Public subnets: `4 environments × 1 Edge VPC × 3 AZs = 12`.
- Private subnets: `4 environments × 3 private-tier VPCs × 3 AZs = 36`.
- Internet Gateways: up to one per Edge VPC, or 4, if each Edge VPC needs direct internet connectivity.
- NAT Gateways: up to one per AZ in each egress VPC, or 12 if each environment's Edge VPC is also its egress VPC and uses three AZs.
- Transit Gateway attachments: 16 if every VPC attaches to one appropriate Transit Gateway; the number of Transit Gateways and route tables remains a separate design choice.

These illustrative totals are **not the environment's actual counts**. Shared networking, centralized egress, additional subnet tiers, or different AZ requirements change them. With centralized egress, application VPC routes should traverse the Transit Gateway and required inspection path to the egress VPC; a route table cannot target a NAT Gateway in another VPC directly. Endpoint totals also depend on service count and AZ placement and must be calculated from the endpoint inventory.

## 5. Foundational Account Services and EKS Roadmap

### Amazon ECR

- Use separate repositories or clear ownership boundaries for application images.
- Enable immutable tags for release repositories so an existing release tag cannot be silently overwritten. Use unique build tags and deploy by digest for exact identity.
- Enable image scanning according to the organization's vulnerability-management process and define remediation ownership.
- Apply least-privilege repository policies and lifecycle rules that retain approved releases while cleaning up eligible untagged/old images.
- Encrypt repositories and audit image pushes and pulls.

### Amazon S3

- Block public access unless a documented, approved use case requires otherwise.
- Enable versioning for buckets that need recovery or audit history.
- Encrypt at rest with an approved key policy; use SSE-KMS when key-level access control and audit are required.
- Define lifecycle transitions and expiration only after data-retention, recovery, and legal requirements are understood.
- Restrict bucket and key policies, use TLS, log access where required, and consider Object Lock for records with retention requirements.
- Separate application data, Terraform state, audit logs, and artifacts into appropriately controlled buckets/accounts.

### IAM baseline

- Federate human access through the organization's identity provider and grant roles through permission sets or groups.
- Use workload roles and short-lived credentials; avoid embedded access keys.
- Apply least-privilege policies, permission boundaries and organization-level guardrails where appropriate.
- Separate administration, deployment, read-only audit, and incident-response responsibilities.
- Protect the root user, enable strong authentication, alert on sensitive IAM changes, and continuously review unused or excessive permissions.

### EKS placement

In the proposed topology, EKS worker nodes and application workloads belong in private **App-zone** subnets in at least the required AZs. Public ingress should terminate at the Edge layer and route only approved application traffic inward. The cluster should use private API access where operationally supported, scoped security groups, and private endpoints for required AWS APIs.

Grant Pods only the AWS permissions they need using EKS Pod Identity or IAM Roles for Service Accounts (IRSA), with narrowly scoped trust policies. Use the existing network routes, VPC endpoints, KMS controls, registry policies, logging, and identity baseline. Keep cluster administration separate from application deployment. Validate subnet IP capacity for nodes, Pods, and load balancers before creating the cluster.

## 6. Data Tier and Safe Model Progression

The data products serve different access patterns; each should have an owner, backup/recovery plan, access policy, and environment boundary.

| Technology | Example role | Key design consideration |
|---|---|---|
| **MySQL** | Transactional records that need relational constraints and consistent updates. | Protect writes and backups; application services should own their schemas and transactions. |
| **Valkey** | Low-latency cache, session, or short-lived coordination data. | Treat it as ephemeral unless persistence and recovery are deliberately configured; never use cache as the only financial record. |
| **Neo4j** | Relationship and graph queries, such as connected fraud signals. | Restrict access to sensitive relationship data and govern retention and data lineage. |
| **MongoDB** | Flexible documents or event-derived read models. | Define consistency, schema evolution, backup, and rebuild procedures. |

### Shadow testing to canary deployment

Keep shadow traffic and production traffic isolated at both the identity and data layers:

1. Run the shadow model in a sandbox or dedicated non-production account and use synthetic, masked, or explicitly authorized read-only copies of production inputs.
2. Give the shadow model separate database endpoints, credentials, encryption keys, queues/topics, and write targets. Do not point its ConfigMap or secret reference at production write connections.
3. In shadow mode, mirror only the minimum permitted input and compare predictions without allowing the model to influence a live decision. Record results in an isolated shadow store.
4. For a canary, use a controlled production rollout with approved routing/feature flags and a defined traffic percentage. Give the canary a production-scoped identity only for the specific authorized operations it needs.
5. Keep canary writes isolated where possible; otherwise use the same validated application transaction path and enforce idempotency, audit, and rollback controls. Never create parallel untracked writes to the ledger.
6. Monitor error rates, latency, decision quality, and data leakage. Stop or roll back automatically when agreed thresholds are exceeded.
7. Promote model versions through review, record the artifact and configuration versions, and remove temporary shadow access and data on schedule.

ConfigMaps should contain non-secret settings only. Database passwords and tokens belong in Vault or an approved secret manager, with environment-specific roles and rotation. A different ConfigMap alone is not sufficient isolation if the workload can still reach or authenticate to production databases.

## 7. Observability, Security, and Governance

Terraform establishes resources and policies; continuous detection, auditing, and response are needed after apply. These AWS services complement one another and do not replace preventive controls or an incident-response process.

| Service | Continuous role | Important limitation |
|---|---|---|
| **AWS CloudTrail** | Records supported AWS API activity, including who made infrastructure changes and when. Centralize and protect trail logs. | It records events; it does not by itself prevent an authorized but harmful change. |
| **Amazon GuardDuty** | Analyzes supported signals for suspicious activity and potential threats in AWS accounts and workloads. | Findings require triage, response automation, and tuning. |
| **AWS Config** | Records resource configuration changes and evaluates resources against selected rules or conformance packs. | Coverage depends on enabled resource types, rules, region/account setup, and remediation design. |
| **Amazon Macie** | Discovers and classifies sensitive data, such as PII, in supported S3 environments and can alert on risky exposure. | It helps discover and monitor data; it is not a general-purpose automatic redaction service. |
| **Amazon Comprehend** | Provides NLP capabilities including PII detection for supported text workflows. It can be used as part of a controlled detection/redaction pipeline. | It is not a continuous infrastructure auditor; validate language, data types, accuracy, latency, and cost for the use case. |

### PII protection and log governance

- Prevent unnecessary PII collection. Prefer opaque identifiers and structured logs over payment credentials or full customer records.
- Redact or tokenize sensitive fields in the application or a controlled ingestion pipeline **before** logs reach broad-access destinations.
- If Comprehend is used to detect and redact PII, call it in a designed processing workflow with validation, failure handling, access controls, and a policy for uncertain results. Do not assume it automatically cleans all logs or S3 objects.
- Use Macie findings to locate and prioritize sensitive S3 data. A separate remediation workflow must quarantine or restrict the affected object, redact or remove sensitive content according to policy, preserve required audit evidence, and verify the resulting object before restoring normal access.
- Encrypt data in transit and at rest, limit log and bucket access, define retention and deletion schedules, and audit reads of sensitive datasets.
- Centralize CloudTrail, GuardDuty, and AWS Config findings in protected security accounts; alert, assign owners, and track remediation to closure.
- Test redaction with representative data and maintain evidence of retention, access, and response controls.

## Implementation Inputs to Confirm

Before treating this reference design as an implementation blueprint, confirm:

- AWS account IDs/OU layout and whether environments have dedicated accounts.
- AWS Regions, required AZ count, CIDR blocks, corporate/on-premises connectivity, and growth assumptions.
- Whether Edge, App, Data, and Ops are separate VPCs, subnet tiers, or a mixture.
- Transit Gateway ownership, route-table segmentation, inspection paths, and approved ingress/egress flows.
- Exact NAT Gateway strategy and the VPC endpoint service inventory by VPC and AZ.
- Terraform module interfaces, state backends, locking, role assumptions, workspace policy, and CI approval gates.
- Data classification, retention, recovery objectives, model traffic policy, and regulatory controls.
