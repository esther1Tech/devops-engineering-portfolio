# Kubernetes Application Deployment

## Introduction

Kubernetes is a container orchestration platform. It runs and manages containerized workloads across a cluster of machines, using declarative configuration to describe the desired state of an application. Kubernetes controllers continuously work to make the actual state match that desired state.

## 1. Why Kubernetes in the Application Lifecycle?

Docker images package an application and its runtime dependencies. Kubernetes provides a platform to run those images reliably and manage them as application workloads:

- **Declarative deployments:** Teams describe the desired number of replicas, image version, ports, and configuration in version-controlled manifests. Kubernetes applies and maintains that state.
- **Self-healing:** Controllers can replace failed containers and reschedule workloads when a node becomes unavailable, subject to available cluster capacity and application configuration.
- **Scaling:** Workloads can be scaled by changing replica counts or by using autoscaling when metrics and policies are configured.
- **Service discovery and networking:** Services provide stable in-cluster network endpoints for changing sets of Pods. Ingress or a Gateway can provide external HTTP routing when configured with the required controller.
- **Safer releases:** Rolling updates can replace application replicas gradually. Readiness checks help prevent unready Pods from receiving Service traffic, and a Deployment can be rolled back to a prior revision.
- **Configuration separation:** ConfigMaps and Secrets allow runtime configuration to be managed separately from the image. Secrets still require appropriate access controls and secure cluster configuration.
- **Resource management:** Requests and limits help Kubernetes schedule workloads and constrain their CPU and memory consumption.
- **Consistent operations:** The same workload definitions can be promoted through development, staging, and production, with environment-specific values supplied separately.

Kubernetes does not remove the need for application testing, monitoring, secure image builds, or operational planning. It also adds operational complexity, so its benefits should justify managing a cluster and its supporting services.

### Typical deployment flow

```text
Code change
    |
    v
CI builds, tests, and scans a versioned container image
    |
    v
Image is published to a container registry
    |
    v
Kubernetes manifests reference the approved image
    |
    v
Cluster starts and manages the application Pods
    |
    v
Monitoring and operational feedback inform future changes
```

## 2. Kubernetes Objects for an Application

Kubernetes objects are records of the desired state of a cluster resource. The objects needed depend on the application: a stateless internal API needs a different set from a public website or a stateful service.

| Object | What it does | When it is needed |
|---|---|---|
| **Pod** | Runs one or more tightly coupled containers and provides their shared runtime environment. | Every application container runs inside a Pod. Usually, create Pods through a controller rather than managing individual Pods directly. |
| **Deployment** | Manages a replicated, stateless application workload, including replacement Pods and rolling updates. | The usual choice for APIs, web applications, and workers that do not require stable per-instance identity or storage. |
| **ReplicaSet** | Maintains the requested number of matching Pods. | Normally created and managed by a Deployment; it is rarely authored directly for an application. |
| **Service** | Provides a stable network endpoint and load-balances traffic to matching Pods. | Needed when other workloads or clients must reach a changing set of Pods through a stable cluster address. |
| **ConfigMap** | Supplies non-sensitive configuration as environment variables or mounted files. | Use when configuration should be managed separately from the container image. Do not put credentials or other sensitive values here. |
| **Secret** | Supplies sensitive values such as credentials or tokens to workloads. | Use when a workload needs secret data. Enable appropriate encryption and access controls; a Secret object alone does not guarantee secure storage. |
| **Namespace** | Groups resources and provides a scope for names and some access-control and policy boundaries. | Useful for separating teams, applications, or environments within a shared cluster; often unnecessary to create one for a single small example. |
| **Ingress or Gateway resources** | Describe external HTTP(S) routing to Services. | Needed when the application must be reachable from outside the cluster through HTTP(S). A compatible controller or Gateway implementation must also be installed. |
| **PersistentVolumeClaim (PVC)** | Requests persistent storage from the cluster. | Needed by workloads that must retain data beyond a Pod's lifecycle. Stateless applications generally should not store durable data in their container filesystem. |
| **ServiceAccount and RBAC resources** | Provide workload identity and control which Kubernetes API actions that identity can perform. | Needed when an application or operator must access the Kubernetes API. Grant only the permissions required. |

### Common baseline for a stateless web application

A typical stateless web application commonly uses:

1. A **Deployment** to run and update replicas from an image in a registry.
2. A **Service** to give those replicas a stable in-cluster endpoint.
3. A **ConfigMap** and/or **Secret** when the application requires externalized runtime configuration.
4. An **Ingress** or **Gateway** when external HTTP(S) access is required.

The Deployment's Pod template should also define appropriate resource requests and limits, startup/readiness/liveness probes where suitable, and a non-root security context when the image supports it. A dedicated Namespace may be appropriate for environment or team boundaries. These settings are fields within workload configuration rather than standalone objects.

For a stateful workload, a PVC may also be required, and a StatefulSet may be more appropriate than a Deployment when instances need stable identities or per-replica storage. In many production architectures, databases and other stateful dependencies are managed by dedicated services rather than deployed with the application.

### Object relationships

```text
Ingress / Gateway (optional external HTTP routing)
                    |
                    v
Service (stable network endpoint)
                    |
                    v
Deployment (desired replicas and rollout management)
                    |
                    v
ReplicaSet (maintains replica count; managed by Deployment)
                    |
                    v
Pods (run the application container image)
        |                     |
        v                     v
ConfigMap / Secret       PVC (optional persistent storage)
```

Use labels and selectors consistently so that a Service targets the intended Pods. Store manifests in version control, review changes, and deploy immutable image versions (preferably pinned by digest) so the running application can be traced to the artifact that passed CI.
