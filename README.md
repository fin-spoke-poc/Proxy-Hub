# Proxy-Hub

This repository is the local enforcement layer for GitHub Org Rulesets when the authoritative workflow logic is owned in another GitHub org.

For this POC, Proxy-Hub exists so project repositories such as Container-App can be governed by required workflows without copying SecOps or Platform logic directly into every application repository.

## Purpose

The core idea is simple:

- upstream teams own the real workflow logic in their own GitHub orgs
- this repo hosts thin proxy workflows in the project governance boundary
- GitHub Org Rulesets target the proxy workflows from this repo
- the proxy workflows delegate to pinned upstream reusable workflows
- audit evidence records both the proxy version and the upstream source version

This allows the project to enforce centrally managed controls while preserving clear ownership boundaries.

## Why This Repo Exists

The governance docs define a hub-and-spoke model with Security-owned, Platform-owned, and BU-owned workflows. In practice, this POC needs a local repository that can act as the ruleset attachment point for workflows whose source of truth lives elsewhere.

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

## Recommended Repository Shape

This repo is currently a stub. The intended structure is:

```text
Proxy-Hub/
├── .github/
│   └── workflows/
│       ├── secops-attest-proxy.yml
│       ├── secops-scan-proxy.yml
│       ├── platform-lint-proxy.yml
│       ├── platform-policy-proxy.yml
│       └── proxy-validation.yml
├── manifests/
│   └── workflow-map.yaml
├── docs/
│   ├── promotion-model.md
│   └── break-glass.md
└── README.md
```

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
			image_name:
				required: true
				type: string
			image_digest:
				required: true
				type: string

jobs:
	attest_and_sign:
		uses: upstream-secops-org/security-hub/.github/workflows/attest-sign.yml@v1.2.3
		permissions:
			id-token: write
			contents: read
			attestations: write
		with:
			image_name: ${{ inputs.image_name }}
			image_digest: ${{ inputs.image_digest }}
		secrets: inherit
```

The wrapper must stay thin. If more than a small amount of control logic is accumulating here, ownership boundaries are drifting.

## Workflow Mapping And Versioning

Proxy-Hub should provide a clear mapping between local proxy versions and upstream workflow versions.

Suggested mapping file:

```yaml
workflows:
	secops-attest-proxy:
		owner: SecOps
		proxy_ref: v1
		upstream_repo: upstream-secops-org/security-hub
		upstream_workflow: .github/workflows/attest-sign.yml
		upstream_ref: v1.2.3
	platform-lint-proxy:
		owner: Platform
		proxy_ref: v1
		upstream_repo: upstream-platform-org/platform-hub
		upstream_workflow: .github/workflows/lint.yml
		upstream_ref: v1.4.0
```

Recommended release model:

1. Upstream team releases or approves a new workflow version.
2. Proxy-Hub updates the pinned upstream reference in a PR.
3. Proxy validation runs against canary repositories or evaluate-mode rulesets.
4. After validation, Proxy-Hub promotes a stable tag such as `v1`.
5. Project rulesets continue referencing the stable proxy tag, not `main`.

This follows the standardization guidance in the project docs and limits blast radius.

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

## Success Criteria For This POC

Proxy-Hub is successful when the project can demonstrate all of the following:

- project rulesets can enforce workflows through this repo
- the actual workflow logic still lives in upstream owner-controlled repos
- Container-App cannot modify or bypass required security and platform checks
- proxy versions can be promoted and rolled back independently of app repos
- audit records clearly show proxy ref, upstream ref, and resulting evidence
- attestation and signing outcomes remain visible to downstream deployment controls

## Alignment To Project Docs

This repo supports the patterns documented in:

- [Org governance](../Docs/0001-org-governance.md)
- [Standardization](../Docs/0002-standardization.md)
- [Attestation](../Docs/0003-attestation.md)
- [Observability](../Docs/0003-observability.md)
- [Unified GitOps](../Docs/0004-unified-gitops.md)
- [Quality thresholds](../Docs/0006-qualitygates/thresholds.md)

## Recommended Next Step

The first useful implementation of this repo should be very small:

1. Add one SecOps attestation proxy workflow.
2. Add one Platform lint proxy workflow.
3. Add `manifests/workflow-map.yaml` to track upstream mappings.
4. Protect the repo and tag the first stable proxy release as `v1`.
5. Point the POC rulesets for Container-App at the proxy workflows in this repo.