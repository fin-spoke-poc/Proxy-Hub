# Proxy-Hub

This repository is the local enforcement layer for GitHub Org Rulesets when the authoritative workflow logic is owned in another GitHub org.

For this MVP, Proxy-Hub exists so project repositories such as Container-App can be governed by required workflows without copying SecOps or Platform logic directly into every application repository.

## Purpose

The core idea is simple:

- upstream teams own the real workflow logic in their own GitHub orgs
- this repo hosts thin proxy workflows in the project governance boundary
- GitHub Org Rulesets target the proxy workflows from this repo
- the proxy workflows delegate to pinned upstream reusable workflows
- audit evidence records both the proxy version and the upstream source version

This allows the project to enforce centrally managed controls while preserving clear ownership boundaries.

## Why This Repo Exists

The governance docs define a hub-and-spoke model with Security-owned, Platform-owned, and BU-owned workflows. In practice, this MVP needs a local repository that can act as the ruleset attachment point for workflows whose source of truth lives elsewhere.

Proxy-Hub is that attachment point.

It should be treated as a controlled projection of upstream governance workflows, not as a new source of business logic.

## Role In This Project

Within this workspace, the intended relationship is:

- Container-App is the spoke application repo under test.
- SecOps is the upstream source repo for security workflows such as scanning, attestation, and signing.
- Proxy-Hub is the local repository referenced by GitHub Org Rulesets for enforcement in this project boundary.

High-level flow:

```text
Container-App PR or merge
 -> GitHub Org Ruleset
 -> required workflow in Proxy-Hub
 -> pinned reusable workflow in upstream org repo
 -> status and evidence returned to Container-App
```

This preserves upstream ownership while still giving the project a stable, governable local control plane.

## Design Goals

This repo should satisfy the following goals:

- enforce required workflows through a project-local repository
- keep upstream Security and Platform teams as owners of workflow behavior
- avoid duplicating workflow logic across app repositories
- version and promote governance changes safely
- provide auditable mapping from proxy workflow to upstream source workflow
- preserve attestation, signing, and ruleset evidence end to end

## What Belongs Here

This repository should only contain thin governance wrappers and supporting metadata.

Expected contents:

- proxy workflows that can be targeted by GitHub Org Rulesets
- pinned references to upstream reusable workflows
- version mapping metadata from proxy tags to upstream workflow refs
- validation workflows to test proxy behavior before promotion
- release notes or changelog entries for governance changes
- documentation for promotion, rollback, and break-glass usage

## What Must Not Belong Here

This repo should not become a second workflow product.

Do not put the following here unless there is a very specific governance reason:

- application business logic
- custom scans that diverge from upstream source repos
- duplicated copies of upstream security logic
- long-lived credentials or manually managed secrets
- BU-specific build and test logic

If the real implementation belongs to SecOps, Platform, or a BU shared-services hub, it should stay there.

## Current Repository Shape

The repository now contains the first working proxy workflow scaffold:

```text
Proxy-Hub/
├── .github/
│   └── workflows/
│       ├── secops-secret-scan-proxy.yml
│       ├── secops-container-scan-proxy.yml
│       ├── secops-attest-sign-proxy.yml
│       ├── validate-proxy-workflows.yml
│       └── release.yml
├── manifests/
│   └── workflow-map.yaml
├── docs/
│   └── promotion-model.md
└── README.md
```

For this MVP, only the SecOps proxy workflows are implemented because the upstream Security hub exists in the workspace. Platform proxy workflows can be added later when the Platform workflow hub is available.

## Proxy Workflow Pattern

Each enforced control should be represented by a thin wrapper workflow in this repo.

The wrapper should:

- define the project-local workflow entrypoint used by the ruleset
- call the authoritative upstream reusable workflow
- pin the upstream reference to a reviewed tag or commit SHA
- pass through only approved inputs and permissions
- emit local metadata showing which proxy version called which upstream version

Example pattern:

```yaml
name: SecOps Attestation Proxy

on:
 workflow_call:
  inputs:
   subject_name:
    required: true
    type: string
   subject_digest:
    required: true
    type: string

jobs:
 attest_and_sign:
  uses: fin-hub-poc/SecOps/.github/workflows/attest-sign.yml@main
  permissions:
   attestations: write
   contents: read
   id-token: write
   packages: write
  with:
   subject_name: ${{ inputs.subject_name }}
   subject_digest: ${{ inputs.subject_digest }}
   registry_provider: ecr
   aws_role_to_assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
  secrets: inherit
```

The wrapper must stay thin. If more than a small amount of control logic is accumulating here, ownership boundaries are drifting.

Once SecOps publishes the first governed `v1` tag, replace `@main` with `@v1` and then publish the first Proxy-Hub release so rulesets can move to the floating major tag.

## Current Workflow Catalogue

The repository now includes the following workflows:

- `secops-secret-scan-proxy.yml`: thin wrapper around the upstream SecOps secret scan workflow.
- `secops-container-scan-proxy.yml`: thin wrapper around the upstream SecOps container scan workflow, including optional pass-through inputs for GHCR, ACR, ECR, and GAR authentication.
- `secops-attest-sign-proxy.yml`: thin wrapper around the upstream SecOps attestation and signing workflow, including optional pass-through inputs for GHCR, ACR, ECR, and GAR authentication.
- `validate-proxy-workflows.yml`: pull request workflow that lints the proxy workflows and validates workflow-map plus version pinning before merge to `main`.
- `release.yml`: manual release workflow that creates immutable semantic version tags and updates the floating major tag.

## Stable Consumer Surface

Proxy-Hub keeps the current public proxy names stable even as SecOps evolves internally.

- Application teams continue calling the same proxy workflow names.
- Registry-specific behavior is selected through optional inputs like `registry_provider`, `registry_server`, and cloud OIDC identifiers.
- SecOps can split internal implementation into separate `attest.yml` and `sign.yml` workflows without forcing downstream repos to change their job graph.

## Workflow Mapping And Versioning

Proxy-Hub should provide a clear mapping between local proxy versions and upstream workflow versions.

Suggested mapping file:

```yaml
workflows:
 secops-attest-sign-proxy:
  owner: Proxy-Hub
  proxy_ref: main
  upstream_repo: fin-hub-poc/SecOps
  upstream_workflow: .github/workflows/attest-sign.yml
  upstream_ref: main
```

The actual mapping is now maintained in `manifests/workflow-map.yaml`.

The proxy workflows are currently mapped to `fin-hub-poc/SecOps` as the upstream SecOps workflow hub.

Recommended release model:

1. During bootstrap, Proxy-Hub tracks `@main` and keeps `workflow-map.yaml` aligned with that reality.
2. Upstream team releases or approves a new workflow version such as `v1`.
3. Proxy-Hub updates the pinned upstream reference in a PR.
4. `validate-proxy-workflows.yml` runs before merge to confirm the proxy files and `workflow-map.yaml` stay aligned.
5. After validation, Proxy-Hub promotes a stable tag such as `v1`.
6. Project rulesets continue referencing the stable proxy tag, not `main`.

This follows the standardization guidance in the project docs and limits blast radius.

See `docs/promotion-model.md` for the promotion details.

## Ruleset Usage Model

The intended enforcement split for this project is:

- Security rulesets target Proxy-Hub workflows that delegate to SecOps-owned workflows.
- Platform rulesets target Proxy-Hub workflows that delegate to Platform-owned workflows.
- BU build and test workflows remain outside Proxy-Hub and are called directly by application repositories.

This keeps required workflows separate from reusable application workflows.

## Security Model

Proxy-Hub is part of the governance chain, so it needs tighter controls than a normal application repo.

Required controls:

- branch protection and PR review
- signed commits or equivalent repository hardening
- pinned upstream refs, never floating `main`
- least-privilege workflow permissions
- no duplicated signing keys or security secrets stored locally
- break-glass handled through temporary ruleset bypass, not permanent workflow edits

The proxy should transmit trust, not recreate trust.

## Observability And Evidence

Proxy-Hub must preserve end-to-end evidence for auditability and debugging.

Each proxy execution should emit or preserve the following fields where possible:

- `repo`
- `app`
- `environment`
- `commit_sha`
- `pr_id`
- `pipeline_stage`
- `ruleset_ids`
- `proxy_repo`
- `proxy_workflow`
- `proxy_ref`
- `upstream_repo`
- `upstream_workflow`
- `upstream_ref`
- `run_id`
- `artifact_digest`
- `attestation_result`
- `signature_result`

This is important for answering questions such as:

- which proxy workflow was enforced by the ruleset?
- which upstream workflow version actually ran?
- which artifact digest was signed or rejected?
- was the failure caused by the app, the proxy, or the upstream workflow?

## Operational Guardrails

To keep this repo maintainable, follow these rules:

- one proxy workflow per enforceable control
- no business-unit customization inside proxy workflows
- all upstream references must be explicit and reviewable
- release changes through PRs with a changelog entry or mapping update
- validate new proxy refs in canary or evaluate mode before broad rollout
- rollback by reverting the pinned upstream ref or restoring the prior proxy tag

## Workflow Validation Before Merge

This repo includes a dedicated validation workflow for branch protection.

`validate-proxy-workflows.yml` does three things before merge to `main`:

- runs `actionlint` against the proxy workflow definitions
- verifies that each proxy workflow is listed in `manifests/workflow-map.yaml`
- verifies that proxy `uses:` lines are pinned to versioned upstream refs such as `@v1`, not branches such as `@main`

That gives the project a safe merge gate even before the real upstream repo coordinates are finalized.

## Success Criteria For This MVP

Proxy-Hub is successful when the project can demonstrate all of the following:

- project rulesets can enforce workflows through this repo
- the actual workflow logic still lives in upstream owner-controlled repos
- Container-App cannot modify or bypass required security and platform checks
- proxy versions can be promoted and rolled back independently of app repos
- audit records clearly show proxy ref, upstream ref, and resulting evidence
- attestation and signing outcomes remain visible to downstream deployment controls
- wrapper workflows remain pinned to released upstream versions and are validated before merge

## Alignment To Project Docs

This repo supports the patterns documented in:

- [Org governance](../Docs/0001-org-governance.md)
- [Standardization](../Docs/0002-standardization.md)
- [Attestation](../Docs/0003-attestation.md)
- [Observability](../Docs/0003-observability.md)
- [Unified GitOps](../Docs/0004-unified-gitops.md)
- [Quality thresholds](../Docs/0006-qualitygates/thresholds.md)

## Next Steps

The first useful implementation of this repo is now in place. The next useful steps are:

1. Confirm `fin-hub-poc/SecOps` remains the intended upstream SecOps repo location.
2. Run `release.yml` to publish the first immutable Proxy-Hub release and floating major tag.
3. Point the MVP rulesets for Container-App at the released proxy workflows in this repo.
4. Add Platform proxy workflows once the upstream Platform workflow hub exists.
