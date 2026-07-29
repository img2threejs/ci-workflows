# img2threejs shared workflows

This public repository contains reusable GitHub Actions workflows for repositories in the `img2threejs` organization. Callers own their triggers, repository permissions, version metadata, and release policy; this repository owns the reviewed workflow implementation.

## Pinning and rollback

Callers MUST reference a workflow by the full commit SHA of a reviewed `ci-workflows` commit, never `main` or a moving version tag. Record the human-readable central workflow version in a comment beside the SHA. To roll back, submit a caller PR that restores the previously known-good SHA or local workflow; do not move published tags or edit releases.

## Reusable workflows

- `python-ci.yml`: read-only checkout, configured Python version, then the caller-owned test command.
- `node-quality.yml`: npm install/build plus an allowlisted post-build check. `showcase-safety` preserves the showcase's full-history PR safety check; it requires `checkout-fetch-depth: "0"`.
- `governed-tag-release.yml`: validates an annotated release tag and creates a GitHub Release. It never calculates a version, creates/moves tags, commits source, or edits a changelog.

## Governed release protocol

Release tags are annotated, canonical `v`-prefixed SemVer: `vMAJOR.MINOR.PATCH`, with optional SemVer prerelease/build metadata. A tag must resolve to the current `main` head and its version after `v` must equal the tagged `SKILL.md` version.

The first governed `v1.4.3` release uses `bootstrap-release: true` and no prior baseline, so invalid historic tags are not used for notes. Every later release requires an explicit annotated `previous-governed-tag` that is both an ancestor and an existing GitHub Release. Stable tags create normal releases; tags with prerelease metadata create GitHub prereleases.

Repository rulesets MUST restrict matching release-tag creation, update, and deletion to release maintainers and protect `main`. Those settings authorize a release; workflow checks only verify the resulting immutable state.
