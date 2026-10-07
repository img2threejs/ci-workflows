# img2threejs shared workflows

This public repository contains reusable GitHub Actions workflows for repositories in the `img2threejs` organization. Callers own their triggers, repository permissions, version metadata, and release policy; this repository owns the reviewed workflow implementation.

## Pinning and rollback

Callers MUST reference a workflow by the full commit SHA of a reviewed `ci-workflows` commit, never `main` or a moving version tag. Record the human-readable central workflow version in a comment beside the SHA. To roll back, submit a caller PR that restores the previously known-good SHA or local workflow; do not move published tags or edit releases.

## Reusable workflows

- `python-ci.yml`: read-only checkout, configured Python version, then the caller-owned test command.
- `node-quality.yml`: npm install/build plus an allowlisted post-build check. `showcase-safety` preserves the showcase's full-history PR safety check; it requires `checkout-fetch-depth: "0"`.
- `governed-tag-release.yml`: validates an annotated release tag and creates a GitHub Release. It never calculates a version, creates/moves tags, commits source, or edits a changelog.
- `npm-publish.yml`: validates the release tag and package version, installs dependencies without lifecycle scripts, tests and packs the same working tree, then publishes that exact artifact through npm trusted publishing (OIDC).

## npm release contract

The build job has `contents: read` only. The separate publish job has `id-token: write`,
downloads the tested tarball, checks its SHA512 integrity, and never runs package scripts.
There is no `NPM_TOKEN` fallback.

| Input | Contract |
|---|---|
| `tag` | Release tag to check out; its version must match `package.json` and `version-file`. |
| `tag-prefix` | Default `v`; use `cli-v` to separate installer releases from skill/plugin releases. |
| `package-directory` | Repository-relative npm package root; default `.`. Plugin installers use `cli/`. |
| `version-file` | Repository-relative JSON with top-level `version`, or markdown with one `version: X` line. |
| `node-version` | Node >=22.14; current callers use Node 24. npm must be >=11.5.1. |
| `python-version` | Python version for the caller's tests. |
| `test-command` | Test command executed in `package-directory` after dependency installation. |
| `provenance` | Default `true`; use `false` for private source repositories. |

Absolute paths, `..` segments and paths resolving outside the checkout are refused.
Packages declaring install, prepare, pack or publish lifecycle hooks are refused.
Dependency installation uses `npm ci --ignore-scripts` with a lockfile, otherwise
`npm install --ignore-scripts --no-audit --no-fund`.

Stable versions publish to `latest`; `1.2.3-beta.1` publishes to `beta`.
A numeric first prerelease identifier uses the named channel `prerelease`.
Build metadata does not become a dist-tag.

The registry probe uses the exact name/version endpoint, including scoped names.
Only HTTP 404 permits publication. An existing version must have matching name, version,
declared integrity and downloaded tarball bytes. Auth failures, outages and malformed responses
fail the job. Re-running an identical release does not move its dist-tag backwards.

## Configure npm trusted publishing

Publish the first version manually from the package root; npm requires an existing package
before configuring a trusted publisher. Use an npm account with package write access and 2FA.
For an organization scope, the account also needs that organization's publish permission.

With npm >=11.15, configure the GitHub caller repository and workflow **filename**:

```bash
npm exec --yes --package=npm@11 -- npm trust github @img2threejs/img2 \
  --repo img2threejs/img2 --file publish.yml --allow-publish --yes
```

The equivalent npm website settings are also supported.
Use `--allow-publish` for direct `npm publish`, not a stage-only configuration.
See [npm trust](https://docs.npmjs.com/cli/v11/commands/npm-trust/).

The reusable-workflow calling job must grant `contents: read` and `id-token: write`;
the shared workflow cannot elevate permissions denied by its caller.
For manual tag input, callers pass `${{ inputs.tag || github.ref_name }}`.
Pin `uses: img2threejs/ci-workflows/.github/workflows/npm-publish.yml@...` to a reviewed
full commit SHA, as with the other shared workflows.

The harness package is `@img2threejs/img2`, while its executable remains `img2`.
The base installer and environment installer are `img2threejs` and `img2-environment`.

## Private source and provenance

npm provenance requires **both** a public GitHub source repository and a public npm package.
The private `img2threejs/plugin-environment` source publishes its public installer with
`provenance: false`; the workflow explicitly sets `NPM_CONFIG_PROVENANCE=false`.
Private plugin content stays outside the npm package, and installation still requires Git access.
See [npm trusted publishers](https://docs.npmjs.com/trusted-publishers/).

This workflow publishes public npm packages (`--access public`); private npm publication
is not part of its contract.

## Governed release protocol

Release tags are canonical SemVer after the configured `tag-prefix`. The default prefix is `v`, so the canonical form is `vMAJOR.MINOR.PATCH` with optional SemVer prerelease/build metadata. A tag must resolve to the current `main` head and its version after the prefix must equal the tagged `SKILL.md` version (or the caller-owned `version-file`).

The first governed release uses `bootstrap-release: true` and no prior baseline, so invalid historic tags are not used for notes. Every later release requires an explicit annotated `previous-governed-tag` that is both an ancestor and an existing GitHub Release. Stable tags create normal releases; tags with prerelease metadata create GitHub prereleases.

Repository rulesets MUST restrict matching release-tag creation, update, and deletion to release maintainers and protect `main`. Those settings authorize a release; workflow checks only verify the resulting immutable state.