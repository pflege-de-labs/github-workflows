# 0001. Share CI and release workflows as reusable workflows

* Status: Accepted
* Date: 2026-10-08

## Context

teamster, compactor, tranquila and nats-auth-callout carry copies of the same jobs: Go checks,
a CI image build, a tagged image release with cosign and attestations, Go release binaries, Helm
chart publishing and SecObserve uploads. The copies have drifted. Action pins
differ by a patch. compactor uploads SBOMs to SecObserve without gating, teamster's Trivy step
fails the build. nats-auth-callout signs its charts, the others do not.

`pflege-de/github-workflows` exists, but it lives in the other organisation, is built around that
organisation's GitHub App Renovate and OPA bundles. Most pflege-de-labs repositories are public,
some, such as nats-auth-callout, are private.

## Decision

We will keep reusable workflows (`on: workflow_call`) for pflege-de-labs in
`pflege-de-labs/github-workflows`, a public repository. Any labs repository can call it, private
ones included, without an Actions access level.

* One workflow per concern, so a caller composes only what it uses. SecObserve is its own
  workflow, called after either image workflow, rather than steps duplicated in both.
* What differs between repositories is an input: Go version source, coverage threshold, generate
  command, chart path. What is the same in all of them is not: action pins, :latest ownership,
  fork handling.
* Repository-specific checks stay in the caller: service containers for tests, chart
  assertions, vendored-asset checks, docs branches. Inputs like `test-command` and
  `assert-command` let a caller run them inside the shared job.
* Secrets are declared and passed by name, not `secrets: inherit`, so a template only sees what
  it needs.
* The repository is versioned as a whole with `vX.Y.Z` tags. pflege-de/github-workflows tags
  per family (`oci/vX`), but the templates here are called together: an image output feeds the
  SecObserve input. Callers pin a commit sha with the tag in a comment, as for any other action,
  and Renovate moves them.
* SecObserve reports and never gates, which was compactor's and tranquila's behaviour.

Renovate is not a template: the Renovate GitHub App runs it for the labs repositories, which
replaces the self-hosted workflows teamster and compactor carried.

Alternatives: composite actions cannot hold several jobs, permissions or services, and would
still need a workflow per caller. A template repository copies once and then drifts like today.

## Consequences

* A fix to a pin or a shared step lands once and reaches every caller through a Renovate PR.
* Keyless signatures now name the template, not the caller: the certificate identity is
  `https://github.com/pflege-de-labs/github-workflows/.github/workflows/<template>.yml@<ref>`,
  and the caller is in the certificate's workflow-repository claim. Verification has to check
  both, and any documentation that tells users how to verify a caller's images or binaries has to
  change when that caller migrates.
* A called workflow cannot ask for more permissions than its caller grants, so each caller job
  grants what the template's README section lists.
* teamster's image build stops failing when the Trivy upload fails.
* Migrating each caller is a pull request in that repository.
