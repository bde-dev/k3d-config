# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This repository contains k3d cluster configuration files for various sandbox environments. Each environment demonstrates different Kubernetes technologies (Harbor, Cilium, CloudNativePG, nginx).

## Environment Structure

Each environment folder follows this pattern:

```
environment-name/
├── environment-name.yaml  # k3d cluster config (k3d.io/v1alpha5 Simple kind)
├── ingress.yaml           # Ingress resource for cluster access
├── namespace.yaml         # Namespace (when needed)
├── README.md              # Environment-specific setup instructions
└── *.yaml                 # Additional manifests (deployments, operators, etc.)
```

## Common Commands

```bash
# Create a cluster (from repo root)
k3d cluster create -c <env-name>/<env-name>.yaml

# Delete a cluster
k3d cluster delete <env-name>

# Switch kubectl context
kubectx k3d-<env-name>

# Monitor cluster with k9s
k9s
```

## Tool Management

Tools are managed via `mise.toml`:

- kubectl, k3d, helm, k9s, kubectx
- cilium-cli, cilium-hubble (for Cilium environment)

Run `mise install` to install all tools.

## Port Mappings

Each environment uses a unique port to avoid conflicts when running multiple clusters:

- 4001: nginx
- 4002: cloudnativepg
- 4003: cilium-cni-ingress
- 4004: harbor (HTTPS)
- 4005: otel-demo

## Devcontainer

Uses docker-in-docker to run k3d clusters inside the devcontainer. The `scripts/setup` script runs on container creation to install tools and add Helm repos.

## Environment-Specific Notes

**harbor**: Has a `bootstrap` script that configures proxy cache registries (Docker Hub, GHCR, Quay, k8s registry, MCR) after cluster creation.

**cilium-cni-ingress**: Disables Traefik, ServiceLB, and Flannel to use Cilium as the CNI. Requires manual Cilium installation after cluster creation.

## Commit Messages

Use [Conventional Commits](https://www.conventionalcommits.org/) format:

```
<type>(<optional scope>): <description>

[optional body]

[optional footer(s)]
```

**Types:**

- `feat` - New feature or capability
- `fix` - Bug fix
- `docs` - Documentation changes only
- `style` - Formatting, whitespace (no code change)
- `refactor` - Code restructuring without behavior change
- `perf` - Performance improvements
- `test` - Adding or updating tests
- `build` - Build system or dependency changes
- `ci` - CI/CD configuration changes
- `chore` - Maintenance tasks, tooling, configs
- `revert` - Revert a previous commit

**Scope:** Optional noun describing the section of codebase affected (e.g., `harbor`, `nginx`, `cilium`)

**Breaking Changes:** Add `!` after type/scope or include `BREAKING CHANGE:` in footer

```
feat(harbor)!: change image pull secret format
```
