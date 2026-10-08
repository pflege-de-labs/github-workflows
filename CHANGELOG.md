# Changelog

Notable changes to the reusable workflows, one entry per release tag `vX.Y.Z`. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions follow
[Semantic Versioning](https://semver.org/): a removed or renamed input, output or secret, or a
permission a caller has to grant anew, is a major release.

## [Unreleased]

### Added

* `go-checks`: build, vet, gofmt, tests with an optional coverage gate, module tidiness,
  generated-output drift, golangci-lint, govulncheck and gosec.
* `image-build`: the CI image of a commit, multi-arch, with an SBOM, signed with cosign; fork
  pull requests build without pushing.
* `image-release`: the image of a release tag with semver tags, :latest only for the newest
  stable tag, SBOM and provenance attestations, signed and verified.
* `secobserve-image`: SBOM and Trivy uploads to SecObserve that report without gating.
* `release-guard`: refuses a release tag that is not on `main`.
* `go-binaries`: cross-compiled binaries with SBOMs, a signed checksum file and the GitHub
  release.
* `helm-lint` and `helm-release`: chart lint and render per values file, and OCI publishing to
  `ghcr.io/<owner>/charts` with optional signing.
* `renovate`: self-hosted Renovate with a caller-provided token.
