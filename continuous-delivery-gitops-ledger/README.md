# Continuous Delivery and GitOps for the Ledger Service

## Purpose and Scope

This document describes a reference continuous delivery (CD) workflow for a Ledger service running on Amazon EKS. It connects short-lived feature branches, functional and post-merge testing, a separate Kubernetes manifest repository, Argo CD reconciliation, and environment-specific runtime configuration.

> **Blueprint-specific note:** The training repository, cluster diagrams, and exact Argo CD configuration were not provided. The namespace names and workflow below are a reference design, not a claim that these pipelines or clusters have already been deployed. Adapt repository permissions, promotion gates, cluster topology, and compliance controls to the approved platform design.

The central release principle is: **build an artifact once, identify it immutably, test it, and promote that same artifact through environments by changing reviewed deployment configuration.** A passing pipeline does not by itself guarantee financial correctness; service-level ledger invariants, reconciliation, and operational controls remain essential.

## 1. Trunk-Based Development and Feature Isolation

### Feature branch versus `main`

- A **feature branch** is short-lived and focused on one small change. Developers commit frequently, open a pull request early, and merge as soon as review and automated checks pass. Feature flags can keep incomplete user-facing behavior disabled while code is integrated.
- **`main` is the trunk:** protected, continuously buildable, and the source of approved release candidates. Each commit should pass required tests and produce a traceable artifact.
- Long-lived release or environment branches tend to diverge, accumulate merge conflicts, and make it unclear which changes are deployed. Keep environment differences in reviewed manifests and runtime configuration, not parallel copies of application source. Release tags or immutable image digests identify releases; they do not require long-lived source branches.

### Feature-branch automation and functional testing

An example pre-merge flow is:

1. A developer pushes a feature branch and opens or updates a pull request.
2. CI checks formatting, compiles the Spring Boot application, runs unit tests and static/security checks, then builds a candidate image if those checks pass.
3. CI publishes the candidate under a unique, traceable identifier (and records its digest). Do not use a mutable tag such as `latest` as the deployed identity.
4. An automation identity updates the **manifest repository** with a temporary Functional namespace deployment referencing that candidate digest and pull-request-specific configuration.
5. Argo CD observes the manifest change and reconciles the Functional namespace in the QA cluster. It applies Kubernetes resources; it does not build the image or run the tests.
6. Functional tests exercise the deployed service through its intended API and verify behavior and integration contracts. Results are reported to the source pull request.
7. When the pull request closes, a cleanup workflow removes the preview manifests and Argo CD prunes the temporary resources, subject to safe ownership and pruning policy.

The business goal is to catch behavioral and integration defects against a deployed service **before merge**, while isolating the candidate from other QA tracks and production data. A preview should use synthetic or appropriately masked data, least-privilege service identities, resource quotas, network policies, and unique namespace names. Do not copy financial production data into a functional test namespace without an approved data-handling design.

This workflow requires explicit GitOps support for preview environments, such as a reviewed manifest-repository change and an Argo CD Application or ApplicationSet that watches the relevant preview path. Argo CD does not automatically deploy arbitrary source branches unless it is configured to do so.

## 2. Decoupled Repositories and Pull-Based GitOps

Use two repositories with distinct responsibilities:

| Repository | Contains | Typical writers |
|---|---|---|
| **Source code repository** | Spring Boot source, tests, Dockerfile, and CI definition. | Developers through feature branches and reviewed pull requests. |
| **Manifest repository** | Kubernetes base manifests, environment overlays, Argo CD Applications/ApplicationSets, and image digest references. | Platform/release automation and authorized reviewers through commits or pull requests. |

### Why CI should not apply directly to the cluster

Giving a general-purpose CI runner cluster-admin credentials creates a high-impact credential that can be exposed through compromised dependencies, workflow changes, runner compromise, or overly broad access. Direct `kubectl apply` also bypasses a durable desired-state review in the deployment repository and makes drift and rollback harder to audit.

In a pull-based model, Argo CD inside or authorized for the target cluster reads the manifest repository and reconciles cluster state to the approved Git state. The CI system needs permission to build and publish an image and to propose a manifest change; it does not need general Kubernetes write access. Argo CD should also use scoped project and destination permissions rather than unrestricted cluster-admin access for every application.

### Trigger CD job and manifest updates

When CI must update the manifest repository after a build:

1. CI resolves the image digest produced by the successful build.
2. A dedicated **Trigger CD** job authenticates to the manifest repository using a narrowly scoped, short-lived credential.
3. Prefer a GitHub App installation token or workload-federated identity where supported. If a fine-grained personal access token (PAT) is required, use a bot-owned token limited to the single manifest repository and only the required content/PR permissions; store it in the CI secret store, restrict which trusted workflows can access it, rotate it, and never print it to logs.
4. The job updates only the appropriate preview or environment overlay to the new digest and creates a commit or pull request. Protect the manifest branch with required checks and approvals. Do not expose the write credential to untrusted pull-request code or run untrusted PR scripts in a privileged workflow; validate the repository, commit, image digest, and target environment before writing the manifest change.
5. Argo CD polls or receives a repository notification, fetches the authorized revision, and syncs the configured resources. A webhook can reduce detection latency but does not change the pull-based authorization model.

Every desired-state change is represented in Git history, allowing reviewers to see who changed an image, configuration, or workload policy and when. Git auditability depends on protected branches, signed or attributable commits where required, review policies, and retention; a commit log alone does not prove that the deployed cluster matches Git. Argo CD health and sync status, audit logs, and drift alerts provide that operational evidence.

## 3. Testing Gates and Promotion Across Environments

### After the pull request merges to `main`

1. CI checks out the approved `main` commit, runs the required build and test suites, creates the Ledger image once, scans it, publishes it, and records the image digest and source commit.
2. CI updates the QA manifest configuration to reference that digest. Argo CD deploys it to the QA cluster.
3. Two isolated QA tracks validate the same candidate in parallel:

| Namespace | Purpose | Example checks |
|---|---|---|
| **Integration ns** | Verify service-to-service contracts and dependencies, using controlled test doubles or dedicated test instances where appropriate. | API contracts, database migration compatibility, event publishing/consumption, authentication, and failure handling. |
| **Regression ns** | Run the established suite of previously passing behaviors against the candidate. | Ledger invariants, idempotency, duplicate/out-of-order event handling, balance calculations, authorization, and regression cases. |

Parallel execution shortens feedback time and limits interference between suites. The namespaces should have separate resource quotas, configuration, test data, and write targets. Namespace names alone do not provide isolation: apply Kubernetes RBAC, network policies, service accounts, secrets, and data-store permissions accordingly.

### Build certification

Certify a candidate only when the required CI checks pass, the image scan and policy gates meet the team's thresholds, Argo CD reports the intended QA revision healthy and synced, and both Integration and Regression suites pass against the same immutable image digest. Record the source commit, image digest, test reports, manifest revision, and approvals as release evidence.

### Promotion to STAGE and PROD

- **STAGE / UAT:** The promotion automation updates the STAGE overlay to the certified digest. Argo CD deploys it to the Stage cluster's UAT namespace. Run acceptance, end-to-end, migration, and operational-readiness checks against production-like but appropriately controlled dependencies and data.
- **PROD / Release:** After required release approval, update the PROD overlay to the same certified image digest. Argo CD in the production cluster reconciles the Release namespace. Environment configuration and secrets differ by environment; the application image remains the same.

Promotion should advance a verified artifact, not rebuild from the branch separately for each environment. If the artifact or manifest must change after certification, repeat the relevant gates.

### Post-Deployment Verification

Post-Deployment Verification (PDV) checks that the live rollout works after Argo CD reports the desired deployment synced:

- Confirm the deployment is healthy, replicas are ready, and the rollout completed.
- Run low-risk smoke checks through the production service endpoint: health, authentication/authorization, and safe read or synthetic transaction flows.
- Observe latency, error rates, saturation, dependency health, and Ledger-specific invariants/reconciliation signals.
- Compare results to defined thresholds and the pre-deployment baseline. Alert or halt further rollout when checks fail.
- Follow the incident or rollback procedure: revert the manifest to the previous approved digest or use the defined progressive-delivery strategy, then verify recovery.

Smoke tests should not create or alter real customer financial records. Use a dedicated test tenant/account and synthetic data, or validate safe read-only paths. Observability complements tests; it does not replace transaction controls, reconciliation, or human incident response.

## 4. Immutable Images and Runtime Configuration

### Build once, promote by digest

The CI build produces a versioned container image containing the operating-system userspace, supported Java Runtime Environment (JRE), application JAR, and required runtime libraries. Publish it to the registry and deploy it using its immutable digest, for example `registry.example/ledger@sha256:...`. Retain the source commit and build provenance so the artifact can be traced.

The same image digest can be deployed to DEV, QA, STAGE, and PROD. Environment-specific endpoints, feature settings, resource values, and credentials are supplied at deployment/runtime. Rebuilding an image with different configuration for each environment weakens the guarantee that the tested artifact is the one promoted.

### Keep configuration and secrets out of the image

- **Non-sensitive configuration:** Store environment-specific settings in reviewed manifest overlays and provide them through Kubernetes ConfigMaps or equivalent configuration mechanisms.
- **Secrets:** Store credentials in an approved secrets manager such as HashiCorp Vault or an AWS secret-management service. Use workload identity and scoped access; inject or mount only what the workload needs.
- **SSL/TLS certificates:** Use an approved certificate manager or Vault-backed delivery and Kubernetes TLS Secret integration where appropriate. Protect private keys, automate rotation, and restrict access. Public certificates or trust bundles may be handled differently from private keys.
- **Avoid image and Git leakage:** Do not bake environment files, secret values, private keys, or production endpoints into image layers or commit them to the manifest repository. Deleting a secret in a later image layer does not reliably remove it from earlier layers or build history.

### Image layers versus runtime mounts

An image is built from read-only layers and image metadata. A running container adds a writable container layer, but this layer is ephemeral and should not be treated as durable storage. Kubernetes does **not** add an image layer to the Docker image when it mounts configuration. Instead, it exposes ConfigMap, Secret, or Vault-provided data to the container as environment variables or mounted files at runtime, overlaid at the configured path.

For example, the image can contain the Spring Boot JAR and default non-sensitive configuration. A Kubernetes Deployment can mount a ConfigMap at an external configuration path or set environment variables, while a Vault agent/CSI integration or external-secrets controller provides secret files/values. The application reads those values at startup or through a supported reload mechanism. Rollouts may be needed when environment variables change; verify the chosen mechanism's update behavior.

Keep configuration schema versioned and reviewed, validate that required keys exist, and ensure the service fails safely when critical configuration or secrets are unavailable. Argo CD deploys declarative references and Kubernetes resources; secret material should be obtained through the approved secret-delivery integration rather than stored as plaintext in Git.

## Example Release Flow

```text
Feature branch
    |
    +-- CI: build/test/scan -> immutable candidate image
    |                              |
    |                    manifest update for preview
    |                              |
    |                    Argo CD -> QA Functional ns
    |                              |
    |                    functional tests -> PR checks
    |
    +-- reviewed PR merged to main
                   |
            CI build once, record digest
                   |
       manifest repo references that digest
                   |
          Argo CD -> QA Integration + Regression namespaces
                   |
              certify candidate
                   |
          promote same digest to STAGE/UAT
                   |
              approval and checks
                   |
          promote same digest to PROD/Release
                   |
                   PDV -> observe / alert / recover
```

## Operational Controls to Confirm

- Source and manifest repository owners, branch protections, reviewer groups, and bot identity permissions.
- Argo CD project scopes, allowed source repositories, destination clusters/namespaces, sync/prune policy, and credential management.
- Namespace provisioning and cleanup policy for temporary functional environments.
- QA test data strategy, database/event isolation, and Ledger invariants used for certification.
- Image registry retention, scanning thresholds, signing/provenance, and digest promotion policy.
- Stage and production approval requirements, rollout/rollback procedure, PDV thresholds, and incident ownership.
- Vault/secret-manager integration, certificate issuance and rotation, and configuration reload behavior.
