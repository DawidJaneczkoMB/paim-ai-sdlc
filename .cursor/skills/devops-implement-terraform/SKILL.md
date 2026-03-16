---
name: implement-terraform
description: Use when creating or modifying Terraform modules and provisioning cloud infrastructure — covers context discovery, module implementation with safety guardrails, architect escalation for topology and service selection decisions, cost estimation, and plan validation before apply. Includes reference patterns for CI/CD pipelines, Kubernetes deployments, and observability stacks that complement Terraform infrastructure.
---

# Terraform Implementation Workflow

Create or modify Terraform modules and provision cloud infrastructure following established IaC patterns and safety guardrails. Ensures consistent resource configuration with proper naming, tagging, state management, and cost estimation before any infrastructure changes are applied.

Respects existing project conventions, validates all changes through `terraform plan`, and escalates architectural decisions (VPC design, service selection, multi-region) to the architect subagent.

## Agent

This skill is used by the `devops-engineer` agent. When infrastructure tasks require Terraform work, spawn the `devops-engineer` agent (Task tool with `subagent_type: "devops-engineer"`).

## When to Use

- Provisioning new cloud infrastructure with Terraform
- Modifying existing Terraform modules or resources
- Adding resources to an existing IaC codebase
- Migrating manual infrastructure to Terraform

## Required Skills

Load and follow before starting:

- `implementing-terraform-modules` — module patterns, state management, safe infrastructure changes
- `technical-context-discovery` — project conventions and existing Terraform patterns

## Workflow

### 1. Context

Follow `technical-context-discovery` to identify existing Terraform setup.

Additionally, always:

- Check `*.instructions.md` → project-specific conventions
- Analyze existing Terraform modules and state configuration
- Discover environment organization (workspaces, Terragrunt)

### 2. Implementation

Follow `implementing-terraform-modules` for:

- Module structure and interfaces
- Resource configurations with proper naming and tagging
- Variable definitions with validation
- State backend configuration

**Guardrails:**

- Always run `terraform plan` before any apply
- Never suggest `terraform apply -auto-approve` for production
- Ensure remote state is configured before applying
- Flag resources with significant cost impact (>10% increase)

### 3. Architect Consultation

Use the `architect` subagent (Task tool with `subagent_type: "architect"`) when:

- Designing new VPC/network topology
- Selecting between competing cloud services (ECS vs EKS, RDS vs Aurora)
- Implementing multi-region or disaster recovery architecture
- Making decisions with significant cost impact (>10% increase)

Skip for: adding resources to existing modules, updating versions, fixing bugs, adding tags.

### 4. Summary (required output)

Every implementation must end with this summary:

```markdown
## Terraform Implementation Summary

### Current State

- [existing IaC configuration]

### Proposed Configuration

- Provider: [AWS / Azure / GCP]
- Resources: [list of resources to create/modify]

### Variables

| Variable | Type | Required | Description |
| -------- | ---- | -------- | ----------- |

### State Backend

- [remote state configuration]

### Cost Estimate

- [approximate monthly cost for new resources]

### Apply Instructions

1. `terraform init`
2. `terraform plan -out=tfplan`
3. Review plan output
4. `terraform apply tfplan`

### Files

- NEW/MODIFIED: [list of files created or modified]
```

## Related Infrastructure Patterns

These reference files cover infrastructure concerns that complement Terraform provisioning. Load when the implementation touches these areas.

### CI/CD Pipelines

See `references/ci-cd-pipeline-patterns.md` — pipeline design, deployment strategies (rolling/blue-green/canary), OIDC authentication, IaC-specific pipeline structure (plan artifact → approval → apply), and monorepo strategies.

Load when: setting up Terraform pipelines, configuring deployment automation, or implementing GitOps workflows.

### Kubernetes Deployments

See `references/kubernetes-patterns.md` — workload types, resource management (requests/limits/QoS), probes, HPA/KEDA scaling, Helm charts, ingress configuration, security contexts, and production resilience (PDB, anti-affinity).

Load when: deploying applications provisioned by Terraform to Kubernetes, configuring workload resources, or creating Helm charts.

### Observability

See `references/observability-patterns.md` — metrics/logs/traces stack selection, SLO/SLI framework (RED/USE methods), alerting strategy with severity levels, structured logging format, and Prometheus alert templates.

Load when: adding monitoring to Terraform-provisioned infrastructure, defining SLOs, or configuring alerting.

## Connected Skills

- `optimize-cloud-cost` — cost-effective infrastructure designs
- `architecture-design` — architectural decisions before module implementation
- `create-implementation-plan` — planning module rollout across environments
