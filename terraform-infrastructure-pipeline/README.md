# Terraform Infrastructure Pipeline

## Introduction

Terraform is an Infrastructure as Code (IaC) tool. Engineers describe infrastructure in configuration files, and Terraform compares that desired configuration with its state and the real environment to determine what needs to change.

The workflow below helps a team review infrastructure changes before applying them and promote changes safely through development, staging, and production.

## 1. Core Terraform Workflow

The core workflow is **write, initialize, format, validate, plan, review, and apply**:

1. **Write configuration:** Define providers, resources, inputs, outputs, and reusable modules in `.tf` files. Keep secrets out of configuration and source control.
2. **Initialize:** Download the required providers and modules and configure the backend that stores Terraform state.
3. **Format:** Apply Terraform's standard formatting so changes are consistent and easier to review.
4. **Validate:** Check the configuration for syntax and internal consistency.
5. **Plan:** Compare the configuration with the state and target environment. Terraform displays the actions it proposes to take.
6. **Review:** Inspect the plan for unexpected replacements, deletions, security changes, or cost impacts. Obtain the required approval.
7. **Apply:** Apply the reviewed plan to create, change, or remove infrastructure.
8. **Verify:** Check the deployed resources, application health, and monitoring. Investigate errors and update the code and state through Terraform rather than making untracked changes in the console.

```text
Terraform configuration
         |
         v
init -> fmt -> validate -> plan -> review/approve -> apply -> verify
                              |
                              v
                   Terraform state and provider APIs
```

### Common local commands

Run these commands from the Terraform root module for the intended environment:

```bash
terraform init
terraform fmt -check -recursive
terraform validate
terraform plan -var-file="dev.tfvars" -out="dev.tfplan"
terraform apply "dev.tfplan"
```

`terraform init` is normally needed when first setting up a working directory or after backend, provider, or module requirements change. The saved plan lets `apply` use the changes that were reviewed, rather than silently making a new plan. Treat a plan file as sensitive: it can contain sensitive values, so do not commit it or publish it where unauthorized users can access it.

Terraform state maps configuration to real resources and may contain sensitive information. Store it in a secured remote backend with access controls, encryption, and state locking where supported. Do not commit state files to source control or edit them manually. Use the provider lock file (`.terraform.lock.hcl`) to make provider selections reproducible.

## 2. Terraform Development Lifecycle and Branching

Use the team's protected `main` branch as the source of approved infrastructure changes. Develop each change on a short-lived feature branch and promote it through a pull request:

```text
main
  |
  +---- feature/network-update
            |
         Develop
            |
         fmt / validate / tests
            |
         Pull request + plan review
            |
         Approval and merge
            |
         main: approved configuration
            |
         Apply to development -> staging -> production
```

### Branch-to-main process

1. **Create a feature branch** from the latest `main` for one focused infrastructure change.
2. **Develop and test** the Terraform code locally. Run formatting and validation, and use static analysis or policy checks required by the team.
3. **Open a pull request.** CI should initialize with the locked provider versions, run formatting and validation checks, and produce a plan for review without applying changes.
4. **Review the plan and code.** Reviewers check the proposed resource changes, access controls, data handling, cost, and impact on dependent services. Require the appropriate approval for higher-risk environments.
5. **Merge to protected `main`** only after required checks and approvals pass. This makes `main` the reviewed source of truth.
6. **Promote the approved change** through the environments using the team's deployment pipeline. Generate and review an environment-specific plan, then apply it under the correct approval and credentials.

Never apply an unreviewed feature-branch plan to production. Avoid creating a new plan between approval and apply if that would change what was reviewed. If the approved plan is stale or no longer safe to apply, generate a fresh plan and review it again.

## 3. Executing Terraform Across Environments

Each environment can require different resource settings, account or subscription access, and deployment approvals. Keep the infrastructure logic reusable, but isolate each environment's configuration, credentials, and state.

### Recommended structure

For environments that need clear isolation, use separate root configurations and separate state for each environment, with shared modules for common infrastructure:

```text
infrastructure/
├── modules/
│   ├── network/
│   └── application/
└── environments/
    ├── dev/
    │   ├── main.tf
    │   ├── backend.tf
    │   └── dev.tfvars
    ├── staging/
    │   ├── main.tf
    │   ├── backend.tf
    │   └── staging.tfvars
    └── prod/
        ├── main.tf
        ├── backend.tf
        └── prod.tfvars
```

Each environment root can call the same versioned modules while supplying appropriate values. Configure a distinct remote state location and access boundary for each environment. Keep backend credentials out of configuration files; provide authentication through the CI platform's identity mechanism or a secure credential store.

From the appropriate environment directory, a pipeline or engineer can run:

```bash
terraform init
terraform fmt -check -recursive
terraform validate
terraform plan -var-file="dev.tfvars" -out="dev.tfplan"
# Review and approve the plan before applying it.
terraform apply "dev.tfplan"
```

For staging and production, run from their respective directories and use their matching variable files, backend state, credentials, and approval gates. Do not reuse a development state file or development credentials for production.

### Environment and pipeline best practices

- **Use isolated state:** Keep state separate by environment and, where needed, by independently managed system. Configure locking and restrict who can read or change state.
- **Promote the same code:** Promote a reviewed commit through environments; vary environment inputs and access, not the underlying logic without review.
- **Use least privilege:** Give automation only the permissions it needs for the target environment. Prefer short-lived federated identity where available instead of long-lived credentials.
- **Protect production:** Require a reviewed plan and explicit approval before applying production changes. Avoid unrestricted automatic applies.
- **Keep plans tied to reviews:** Apply the exact saved plan that was reviewed when it is still valid. If it is stale, regenerate and review it; protect saved plans because they can expose sensitive information.
- **Pin dependencies:** Constrain Terraform and provider versions, commit `.terraform.lock.hcl`, and upgrade dependencies through reviewed changes.
- **Check before apply:** Run formatting, validation, tests, security scanning, and policy checks. Review destructive or replacement operations carefully.
- **Keep secrets out of Git:** Supply sensitive values through a secret manager or CI's secure variables. Marking a Terraform value sensitive hides some display output but does not remove it from state.
- **Avoid accidental destruction:** Do not use `terraform destroy` as routine cleanup in shared or production environments. Require separate review and approval for destructive operations.
- **Observe and reconcile:** Verify resources after applying and investigate drift. Make intended changes in Terraform so configuration and real infrastructure remain aligned.

Terraform workspaces can be useful for multiple state instances of the same configuration, but they are not by themselves a strong security boundary. For production isolation, separate root configurations and backend state locations with distinct permissions are generally easier to reason about.
