# Enterprise Kubernetes on AWS

## Purpose and Scope

Amazon Elastic Kubernetes Service (EKS) is a managed Kubernetes service for running containerized applications on AWS. This document describes a reference production design for a multi-AZ EKS platform, including its operating model, worker-node pools, scaling behavior, and six production design principles.

> **Blueprint-specific note:** The cluster diagrams and exact configurations referenced in the training materials were not available in the repository. This document explains a production-oriented reference design; it does not claim that a particular cluster, node count, add-on configuration, or monitoring pipeline has already been deployed. Confirm workload requirements, AWS Region, Kubernetes version, subnet and IP capacity, instance types, and operational policies before implementation.

## 1. Why EKS?

EKS provides a managed Kubernetes control plane integrated with AWS networking, identity, logging, and compute. AWS operates the Kubernetes API-server control-plane infrastructure; the platform team still owns cluster configuration, workload design, worker capacity, security, upgrades, and application operations.

### Business and technical benefits

- **Managed control plane:** AWS operates the Kubernetes control-plane service, reducing the burden of operating API servers and etcd directly. Teams still need to plan for control-plane availability, endpoint access, upgrades, and AWS service limits.
- **Resilience across Availability Zones:** EKS can distribute managed control-plane components across multiple AZs. Worker nodes and application replicas should also span AZs so a single-AZ outage does not remove all serving capacity.
- **Elastic capacity:** Kubernetes can add or remove Pods as application demand changes. A node autoscaler can add worker capacity when Pods cannot be scheduled, subject to quotas, instance availability, and subnet IP capacity.
- **AWS integration:** EKS workloads can use VPC networking, IAM workload identities, ECR, CloudWatch, load balancing, and private AWS service endpoints.
- **Consistent application platform:** Teams can deploy services written in different languages and runtimes as containers using common deployment, security, and observability practices.
- **Potential cost efficiency:** Bin-packing, autoscaling, right-sized requests, and appropriate instance purchasing options can improve utilization. EKS is not automatically cheaper: idle capacity, over-provisioning, cross-AZ traffic, logging volume, and data-transfer costs need active management.
- **Delivery agility:** Declarative manifests and Git-driven CI/CD make changes reviewable, repeatable, and easier to roll back than manual host-by-host deployment.

EKS is valuable when the organization needs Kubernetes capabilities and has the operational maturity to secure and run them. It does not remove the need for application reliability engineering or make every workload cheaper or simpler.

## 2. Operating Model: Infrastructure and Applications as Code

Manage the cluster foundation and workloads declaratively:

1. **Infrastructure code** provisions the EKS cluster, VPC attachment and subnets, node groups, IAM roles, security groups, encryption, and required add-ons.
2. **Pull requests** review infrastructure and application changes. CI runs formatting, validation, tests, policy and security checks, and generates a Terraform plan or Kubernetes manifest validation results.
3. **Approved pipelines** apply infrastructure plans and deploy versioned application images/manifests to the target environment. Production changes require designated approvals and least-privilege deployment identities.
4. **Reconciliation tools** such as GitOps controllers can continuously reconcile cluster resources with reviewed Git state. Avoid having multiple controllers and manual processes compete to own the same resource.
5. **Operations** monitor health, capacity, security findings, and deployment results. Emergency changes are documented and reconciled back into code.

Avoid manually configuring worker hosts. Use immutable, supported node images and replace nodes through managed node-group or autoscaling workflows. Keep environment configuration separate from secret values; deliver secrets through a controlled secrets manager and workload identity.

## 3. EKS Topology and Network Capacity

### Control plane and data plane

The **control plane** runs the Kubernetes API server and cluster management components. With EKS, AWS operates this managed control plane in AWS-managed infrastructure, separate from the customer-managed worker subnets. It is not accurate to say the customer deploys EKS control-plane machines into its private subnets.

The **data plane** is the compute that runs Pods: EKS managed node groups, self-managed nodes, or a supported serverless option. In this reference design, worker nodes run in private App subnets across three AZs. Restrict access to the Kubernetes API endpoint using private endpoint access, authorized network paths, and IAM authentication as appropriate for the operational model.

```text
                 AWS Region
  ┌─────────────────────────────────────────┐
  │ AWS-managed EKS control plane           │
  │ Kubernetes API / cluster management     │
  └───────────────────┬─────────────────────┘
                      │ Restricted API access
       ┌──────────────┼──────────────┐
       v              v              v
   AZ A private   AZ B private   AZ C private
   App subnet     App subnet     App subnet
   Core nodes     Core nodes     Core nodes
   MLOps nodes    MLOps nodes    MLOps nodes
       └──────────────┴──────────────┘
              Pods scheduled across AZs
```

Three-AZ placement improves fault tolerance only when capacity, replicas, storage, and dependencies are also available across those AZs. Use topology-spread constraints or pod anti-affinity for critical replicas and ensure that subnet IP address capacity supports nodes, Pods, and load balancers.

### Why keep worker nodes out of public subnets?

This reference design places worker nodes in private subnets to reduce direct exposure and ensure inbound application traffic enters through controlled edge components such as a load balancer or ingress. Nodes do not need public IP addresses to reach ECR or other supported AWS services when the required VPC endpoints and routing are configured.

Private subnets alone are not a security boundary: enforce security groups, network policies where supported, IAM least privilege, endpoint policies, and controlled egress. A node in a subnet with a route to an Internet Gateway is not necessarily reachable from the internet unless it has a public address and permissive rules, but public worker placement is unnecessary for this design and increases exposure risk.

### Capacity planning

Before creating a cluster, calculate per-AZ subnet capacity for node ENIs, Pod IP allocation (including the Amazon VPC CNI configuration), load balancer interfaces, VPC endpoint ENIs, and growth headroom. AWS reserves addresses in every IPv4 subnet. Include expected peak replica counts, rolling-upgrade surge, and node replacement capacity. Ensure account quotas and instance capacity are available in all target AZs.

## 4. Workload Segregation: Core and MLOps Node Groups

Separate compute pools allow the platform to apply different instance types, scaling policies, security boundaries, and cost controls.

| Node group | Example workloads | Scheduling controls |
|---|---|---|
| **Core Services NodeGroup A** | BFF/API, payment, ledger, and shared platform workloads. | Use labels and node affinity or selectors for placement; allocate CPU/memory requests and limits. Apply topology spread for replicated critical services. |
| **MLOps NodeGroup B** | KServe inference workloads, including fraud models that require specialized CPU, memory, or GPU capacity. | Taint the nodes for dedicated workloads, label them by accelerator/type, and require matching tolerations plus node affinity. |

Example configuration:

```text
NodeGroup B taint: workload=mlops:NoSchedule
NodeGroup B label: workload=mlops

MLOps Pod:
  toleration: workload=mlops:NoSchedule
  nodeSelector: workload=mlops
```

A **taint** repels Pods that do not have a matching **toleration**. A toleration only permits scheduling; it does not guarantee placement on the intended nodes. Pair it with a node selector or required node affinity. This prevents ordinary CPU-bound workloads from consuming dedicated GPU nodes, while still allowing platform administrators to deliberately schedule approved workloads there.

Use accelerator resource requests (for example, `nvidia.com/gpu`) and the required device plugin/operator when GPU workloads need them. Keep a fallback/capacity policy for GPU scarcity and ensure autoscaling recognizes the node-group labels, taints, and instance constraints. Taints do not replace IAM, network, or namespace security boundaries.

## 5. Dynamic Scaling Control Loops

Scaling has separate control loops:

- **Horizontal Pod Autoscaler (HPA)** changes the desired number of Pods based on configured resource or custom metrics.
- **Cluster Autoscaler** changes node-group capacity when Pods remain unschedulable because existing nodes cannot satisfy their requests or scheduling constraints.
- **AWS Auto Scaling Group (ASG)** or the EKS managed node group's backing capacity launches or terminates EC2 instances to reach the requested node count.

```text
Traffic / metrics increase
          |
          v
HPA increases desired Pod replicas
          |
          v
Scheduler cannot fit some Pods -> Pods remain Pending
          |
          v
Cluster Autoscaler detects unschedulable Pods
          |
          v
Node group / ASG desired capacity increases
          |
          v
EC2 node launches, joins cluster, becomes Ready
          |
          v
Scheduler places Pending Pods on available nodes
```

The scheduler does not itself scale the ASG. It marks Pods unschedulable when no node satisfies their requirements; Cluster Autoscaler observes that condition and requests more node capacity if a configured group can fit the Pod. Provisioning is not instantaneous, so maintain enough warm capacity for expected burst and startup times. Scaling can be blocked by quotas, AZ capacity, subnet IP exhaustion, taints, affinity, resource requests, or node-group limits.

### PodDisruptionBudget and `minAvailable: 2`

A PodDisruptionBudget (PDB) limits how many matching Pods can be voluntarily disrupted at once, such as during node draining or some upgrade operations. For a critical service, `minAvailable: 2` means at least two matching Pods must remain available during a voluntary disruption. The workload therefore needs more than two healthy replicas to permit a disruption; with only two replicas, draining one may be blocked.

`minAvailable: 2` is a policy choice, not a universal EKS requirement or a guarantee of high availability. Use it when the service's availability objective requires two replicas to remain, and pair it with adequate replicas, readiness probes, topology spreading, and capacity in multiple AZs. PDBs do not prevent involuntary outages, application failures, or loss of an entire AZ, and an overly strict PDB can block maintenance.

## 6. Six Enterprise Design Principles

### 1. High availability

EKS control-plane components are managed by AWS across multiple AZs in the Region. Spread worker nodes and replicas across three AZs, maintain spare capacity, use readiness checks, and make dependencies such as load balancing and storage appropriately resilient. Multi-AZ placement reduces single-AZ risk but does not by itself ensure service availability.

### 2. Disaster recovery

Availability Zones protect against some zonal failures; they are not regional disaster recovery. Regional recovery requires a documented RTO/RPO, infrastructure and cluster recreation plans, image and artifact availability, tested backups or replication for stateful data, DNS/traffic failover, and regular recovery exercises. Cross-AZ nodes alone do not provide regional failover.

### 3. Horizontal autoscaling

Use HPA to add/remove Pods based on validated metrics, and Cluster Autoscaler or another supported node provisioner to adjust compute capacity for unschedulable Pods. Configure requests and limits, scale-down safeguards, PodDisruptionBudgets, and maximum capacity. Test bursts and scale-down behavior; autoscaling does not instantly supply capacity or guarantee the correct workload fits.

### 4. Version upgrades

Upgrade the EKS control plane and add-ons through supported version steps, then roll worker nodes in batches. Test compatibility in lower environments, keep sufficient capacity during rollout, use PDBs and topology spread, and observe error/latency metrics. A rolling strategy can aim for no customer-visible downtime, but zero downtime cannot be guaranteed: incompatible APIs, insufficient capacity, bad health checks, strict PDBs, or application defects can interrupt service. Define rollback and recovery procedures.

### 5. Zero-trust security

- Keep worker nodes in private subnets and restrict control-plane endpoint access.
- Use IAM roles for cluster administration and workload identity (EKS Pod Identity or IRSA) for Pod-to-AWS access; scope permissions to each workload.
- Enforce least privilege with Kubernetes RBAC, network policies where supported, security groups, admission controls, image scanning, and secret-management practices.
- Use encrypted transport and storage, audited identity, short-lived credentials, and controlled egress.
- Treat private networking as one layer of defense, not as proof that a workload or identity is trusted.

### 6. Observability: five-tier matrix

The following is a reference implementation matrix, not a claim about exact configurations already established in a cluster.

| Tier | What to observe | Example implementation |
|---|---|---|
| **Overall infrastructure health** | EC2/node CPU, memory, disk, network, AZ capacity, cluster and AWS service metrics, scaling events. | CloudWatch metrics and Container Insights or a Prometheus-based platform; dashboards and alerts tied to SLOs. |
| **Application health** | Request rate, errors, latency, saturation, dependency health, and business-level outcomes. | Application metrics, structured logs, distributed traces, readiness/liveness/startup probes, and service-level objectives. |
| **Kubernetes events and control signals** | Pod scheduling failures, restarts, evictions, deployment rollouts, node conditions, and autoscaler decisions. | Centralized Kubernetes API audit/events and controller logs with retention, alert routing, and runbooks. |
| **Runtime threat detection** | Unexpected process execution, privilege escalation, suspicious file/network behavior, and policy violations. | Falco rules and alerts forwarded to a monitored security pipeline. Falco detects runtime behavior; it is not a substitute for image vulnerability scanning. |
| **Log management pipeline** | Application, node, control-plane, and security logs with useful metadata and controlled retention. | Fluent Bit or Vector agents collect, parse, redact sensitive fields where possible, and forward logs to an access-controlled backend. Avoid running two collectors for the same source unless their roles are explicitly separated. |

Set log retention and access based on operational and regulatory requirements. Redact sensitive information before broad distribution, protect audit trails from alteration, alert on missing telemetry, and test that responders can query the data they need.

## Operational Readiness Checklist

- Confirm cluster region, Kubernetes version, endpoint access model, and supported add-on versions.
- Verify private subnet capacity, Pod IP strategy, three-AZ node-group capacity, and service quotas.
- Define core and MLOps node labels, taints, tolerations, resource requests, and autoscaling limits.
- Test HPA and node autoscaling with realistic load and node-startup delays.
- Validate PDB, readiness, topology-spread, and rolling-upgrade behavior in a non-production environment.
- Confirm workload IAM, RBAC, network policies, image policies, encryption, and secret delivery.
- Document dashboards, alerts, on-call ownership, backup/restore procedures, and regional disaster-recovery tests.
- Keep all infrastructure and application deployment changes reviewed, automated, and auditable.
