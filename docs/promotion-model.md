# Promotion Model

This repository promotes approved upstream SecOps workflow releases into project-local proxy workflows.

## Principle

Proxy-Hub does not own the security logic. It owns the project-local attachment point and the version pinning strategy.

## Promotion Flow

1. SecOps publishes a reviewed release such as `v1.0.0` and maintains the floating major tag such as `v1`.
2. Proxy-Hub updates the pinned upstream repo and workflow references in the proxy workflows and `manifests/workflow-map.yaml`.
3. `validate-proxy-workflows.yml` must pass before merge to `main`.
4. After merge, `release.yml` creates a new immutable Proxy-Hub tag such as `v1.0.0` and refreshes the floating major tag such as `v1`.
5. GitHub Org Rulesets continue pointing to the Proxy-Hub major tag, not `main`.

## Activation Checklist

- Replace `example-secops-org/secops-workflow-hub` with the actual upstream repo location.
- Confirm the upstream repo exposes the referenced workflow files and release tags.
- Confirm the Proxy-Hub workflow map matches the wrapper workflow `uses:` lines.

## Rollback

If an upstream workflow version causes a regression, revert the pinned upstream ref in Proxy-Hub, merge the change, and publish a new immutable Proxy-Hub release that updates the floating major tag to the last approved state.
