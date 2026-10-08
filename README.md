# github-workflows

Reusable GitHub Actions workflows for the pflege-de-labs repositories: Go checks, container
images, Go release binaries, Helm charts and SecObserve. They were extracted from
teamster, compactor, tranquila and nats-auth-callout. Why they are shaped the way they are is in
[ADR 0001](docs/adr/0001-one-repository-of-reusable-workflows.md).

| Workflow | Does |
| --- | --- |
| [`go-checks`](#go-checks) | build, vet, gofmt, test + coverage, tidy, generated drift, golangci-lint, govulncheck, gosec |
| [`image-build`](#image-build) | CI image of a commit, signed |
| [`image-release`](#image-release) | release image of a tag, signed, attested, verified |
| [`secobserve-image`](#secobserve-image) | SBOM and Trivy upload to SecObserve, never gating |
| [`release-guard`](#release-guard) | refuses a tag that is not on `main` |
| [`go-binaries`](#go-binaries) | release binaries, SBOMs, signed checksums, GitHub release |
| [`helm-lint`](#helm-lint) | lint and render a chart per CI values file |
| [`helm-release`](#helm-release) | publish charts to `ghcr.io/<owner>/charts`, optionally signed |

## Using them

Pin a commit sha with the release tag in a comment, as for any action. Renovate's
`helpers:pinGitHubActionDigests` keeps the pin moving:

```yaml
jobs:
  checks:
    uses: pflege-de-labs/github-workflows/.github/workflows/go-checks.yml@<sha> # v1.0.0
```

* **Permissions.** A called workflow cannot have more than the calling job grants. Each section
  below lists what the calling job needs; set it on the job, not only on the workflow.
* **Secrets.** Pass them by name, as the examples do. No template needs `secrets: inherit`.
* **Concurrency.** Cancelling superseded runs is the caller's choice, so it stays in the caller.
* **Signatures.** A keyless signature names the workflow that made it, which is the template:
  verify with the template's identity and pin the caller with the workflow-repository claim:

  ```bash
  cosign verify \
    --certificate-identity-regexp '^https://github\.com/pflege-de-labs/github-workflows/\.github/workflows/image-release\.yml@' \
    --certificate-github-workflow-repository pflege-de-labs/teamster \
    --certificate-oidc-issuer https://token.actions.githubusercontent.com \
    ghcr.io/pflege-de-labs/teamster:1.2.3
  ```

  `go-binaries` and `helm-release` sign the same way under their own file names.

## go-checks

Permissions: `contents: read`.

| Input | Default | |
| --- | --- | --- |
| `go-version` | `""` | e.g. `stable`; wins over `go-version-file` |
| `go-version-file` | `go.mod` | |
| `test-command` | `go test ./... -coverprofile=coverage.out` | coverage is reported when `coverage.out` exists |
| `coverage-threshold` | `0` | minimum total coverage in percent; `0` only reports |
| `gofmt` | `true` | |
| `modules` | `.` | whitespace-separated module directories that must be tidy |
| `generate-command` | `""` | e.g. `make generate`; empty skips the drift check |
| `generate-diff-args` | `""` | extra `git diff` arguments, e.g. `-I '^\.TH ' -- docs/man` |
| `golangci-lint-version` | `""` | empty skips linting |
| `govulncheck` | `true` | runs the latest govulncheck on purpose |
| `gosec-version` | `v2.29.0` | empty skips gosec |

Tests that need service containers (Postgres, NATS) cannot get them from a reusable workflow.
Keep such a test job in the caller and turn coverage off here with a cheap `test-command`, or
start the dependency from `test-command`.

## image-build

Permissions: `contents: read`, `packages: write`, `id-token: write`.

Tags the image with the short sha, the branch and the PR, never `:latest`. Fork pull requests
build without pushing or signing. The image carries an SBOM attestation, no provenance: that would
add an `unknown/unknown` manifest on GHCR.

| Input | Default | |
| --- | --- | --- |
| `image` | `ghcr.io/<owner>/<repo>` | lowercased |
| `context` | `.` | |
| `file` | `<context>/Dockerfile` | |
| `platforms` | `linux/amd64,linux/arm64` | |
| `version` | `sha` | `VERSION` build arg: `sha`, or `describe` for `git describe` against release tags |
| `describe-match` | `v[0-9]*` | tags `describe` considers |
| `build-args` | `""` | further `KEY=value` lines |
| `sign` | `true` | |

Outputs: `image`, `digest`, `pushed`, `short-sha`.

## image-release

Permissions: `contents: read`, `packages: write`, `id-token: write`. Call it from a workflow
triggered by a `v*` tag push.

Tags `X.Y.Z`, `X.Y` and `X`. `:latest` moves only when the tag is the newest stable one, so
re-releasing an old tag or cutting a pre-release leaves it alone. The image carries SBOM and
provenance attestations; both and the signature are verified before the job ends.

| Input | Default | |
| --- | --- | --- |
| `image`, `context`, `file`, `platforms`, `build-args` | as `image-build` | `VERSION` is the tag |
| `tag-pattern` | `v*` | release tags, for finding the newest stable one |
| `version-check` | `false` | run the image with `--version` and require the tag in the output |

Outputs: `image`, `digest`.

## secobserve-image

Permissions: `contents: read`, `packages: read`. Secret: `SO_API_TOKEN`.

Uploads each platform's SBOM attestation and a Trivy scan of the digest. A failed upload becomes a
warning and a note in the job summary, never a failed run. Skip it for pull requests in the caller,
so short-lived PR branches do not appear in SecObserve.

| Input | Default | |
| --- | --- | --- |
| `image` | required | name without tag |
| `digest` | required | |
| `tag` | required | recorded as the image's name in SecObserve |
| `product-name` | repository name | also the origin service |
| `branch-name` | `github.ref_name` | a release tag is tracked as its own branch |
| `platforms` | `linux/amd64,linux/arm64` | |
| `api-base-url` | `https://webhooks.management.p4e.io/secobserve` | |

## release-guard

Permissions: `contents: read`. Input `branch`, default `main`. Run it first in a release and have
the rest `needs:` it.

## go-binaries

Permissions: `contents: write`, `id-token: write`.

Builds `<name>-<os>-<arch>` with `CGO_ENABLED=0 -trimpath -s -w`, links the tag into
`version-symbol`, writes an SPDX SBOM per binary, and signs `checksums.txt` (which covers
binaries and SBOMs) as a cosign bundle. A tag with a `-` suffix becomes a pre-release.

| Input | Default | |
| --- | --- | --- |
| `name` | required | |
| `package` | required | e.g. `./cmd/teamster` |
| `targets` | `linux/amd64 linux/arm64 darwin/amd64 darwin/arm64` | |
| `version-symbol` | `main.version` | |
| `go-version`, `go-version-file` | as `go-checks` | |

## helm-lint

Permissions: `contents: read`.

| Input | Default | |
| --- | --- | --- |
| `chart` | required | e.g. `charts/teamster` |
| `values-glob` | `ci/*-values.yaml` | relative to the chart; no match lints the defaults |
| `assert-command` | `""` | e.g. `scripts/check-chart-render.sh charts/teamster` |

## helm-release

Permissions: `contents: write`, `packages: write`, `id-token: write` (even with `sign: false`).

Publishes every chart under `charts-dir` whose `Chart.yaml` version has no GitHub release yet, and
creates the release `<chart>-<version>`. Call it on pushes to `main` that touch the charts.

| Input | Default | |
| --- | --- | --- |
| `charts-dir` | `charts` | |
| `helm-version` | `v4.2.3` | |
| `sign` | `false` | sign each chart version that is not signed yet |

## Example: a Go service

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
permissions:
  contents: read
jobs:
  checks:
    uses: pflege-de-labs/github-workflows/.github/workflows/go-checks.yml@<sha> # v1.0.0
    with:
      golangci-lint-version: v2.14.0
  chart:
    uses: pflege-de-labs/github-workflows/.github/workflows/helm-lint.yml@<sha> # v1.0.0
    with:
      chart: charts/my-service
  image:
    needs: checks
    uses: pflege-de-labs/github-workflows/.github/workflows/image-build.yml@<sha> # v1.0.0
    permissions:
      contents: read
      packages: write
      id-token: write
  secobserve:
    needs: image
    if: github.event_name != 'pull_request'
    uses: pflege-de-labs/github-workflows/.github/workflows/secobserve-image.yml@<sha> # v1.0.0
    permissions:
      contents: read
      packages: read
    with:
      image: ${{ needs.image.outputs.image }}
      digest: ${{ needs.image.outputs.digest }}
      tag: ${{ needs.image.outputs.short-sha }}
    secrets:
      SO_API_TOKEN: ${{ secrets.SO_API_TOKEN }}
```

```yaml
# .github/workflows/release.yml
name: Release
on:
  push:
    tags: ["v*"]
permissions:
  contents: read
jobs:
  guard:
    uses: pflege-de-labs/github-workflows/.github/workflows/release-guard.yml@<sha> # v1.0.0
  verify:
    needs: guard
    uses: pflege-de-labs/github-workflows/.github/workflows/go-checks.yml@<sha> # v1.0.0
  image:
    needs: verify
    uses: pflege-de-labs/github-workflows/.github/workflows/image-release.yml@<sha> # v1.0.0
    permissions:
      contents: read
      packages: write
      id-token: write
    with:
      version-check: true
  secobserve:
    needs: image
    uses: pflege-de-labs/github-workflows/.github/workflows/secobserve-image.yml@<sha> # v1.0.0
    permissions:
      contents: read
      packages: read
    with:
      image: ${{ needs.image.outputs.image }}
      digest: ${{ needs.image.outputs.digest }}
      tag: ${{ github.ref_name }}
    secrets:
      SO_API_TOKEN: ${{ secrets.SO_API_TOKEN }}
  binaries:
    needs: image
    uses: pflege-de-labs/github-workflows/.github/workflows/go-binaries.yml@<sha> # v1.0.0
    permissions:
      contents: write
      id-token: write
    with:
      name: my-service
      package: ./cmd/my-service
```

## Developing

Dependencies are updated by the Renovate GitHub App from [renovate.json](renovate.json); nothing
merges itself, since every caller runs what lands here.

Pull requests run [actionlint](https://github.com/rhysd/actionlint) (with shellcheck) over every
workflow. Run it locally with `actionlint`. Templates stay flat in `.github/workflows/` (GitHub
does not look in subdirectories), and keep their scripts inline: the checkout in a called workflow
is the caller's repository, so a script file here would not be on disk.

Renovate bumps `CRANE_VERSION` in `helm-release.yml` but not `CRANE_SHA256` next to it; update
the checksum from the release's `checksums.txt` in the same pull request.

Releases are tags `vX.Y.Z` on `main`. Move the `Unreleased` entries in
[CHANGELOG.md](CHANGELOG.md) under the new version first. Removing or renaming an input, output
or secret, or asking callers for a new permission, is a major release.
