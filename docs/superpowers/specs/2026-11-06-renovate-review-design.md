# Renovate review by Claude: design

Date: 2026-11-06. First caller: pflege-de-labs/teamster; tranquila and the other pflege-de-labs
repositories follow.

## Goal

Renovate opens dependency PRs every Monday. Today a maintainer approves each one by hand. Claude
should review them: approve the uncritical ones so that they merge, and hand the rest to a
maintainer with the label `human-review-required` and a note on what to look at. Once a
maintainer approves a labelled PR, it merges without further work.

Success: a routine minor, patch or digest bump with green CI merges without a person; a major,
a toolchain or runtime image bump, a failing CI or anything Claude is unsure about waits for a
maintainer and says why.

## Constraints

- **Who may merge.** Every default branch requires a code-owner review, and bots cannot be code
  owners (token-broker ADR 0004). The Renovate app bypasses that rule in pull-request mode, so a
  PR *Renovate* merges needs one ordinary approval, which the agent app may give. Claude
  therefore approves; Renovate's auto-merge merges. Claude never merges itself.
- **No stored secrets.** The GitHub token comes from the token-broker (`POST /token/actions`,
  agent app, 1h, this repository only). Claude Platform access comes from workload identity
  federation through the `claude-code-action` federation inputs.
- **The broker only serves `refs/heads/main` and `refs/tags/v*`** (`actionsOidc.allowedRefs` in
  cc-gitops). A `pull_request` run carries `refs/pull/<n>/merge` and gets no token; a
  `pull_request_target` run carries the base branch and does.
- **The agent app cannot write workflow files.** Workflow files are committed by a person.

## Design

### Pieces

| Piece | Where | Committed by |
|---|---|---|
| Reusable workflow `renovate-review.yml` | github-workflows | person (workflow file) |
| Review prompt `prompts/renovate-review.md` | github-workflows | agent |
| Renovate preset `renovate/agent-review.json` | github-workflows | agent |
| ADR 0002, README section, CHANGELOG | github-workflows | agent |
| Caller workflow `renovate-review.yml` | each repository | person (workflow file) |
| `renovate.json` extends the preset, plus repository rules | each repository | agent |
| Federation rule | Claude Console, one per repository | person |
| ADR, docs, CHANGELOG | each repository | agent |

### Flow

1. **Renovate opens or rebases a PR.** The preset turns on auto-merge for every update, so
   GitHub merges as soon as the PR has one approval and the required checks pass. The preset
   labels majors `human-review-required`; a repository adds its own always-human rules (teamster:
   the Go toolchain and the runtime base image).
2. **Caller workflow** on `pull_request_target` (`opened`, `reopened`, `synchronize`) calls the
   reusable workflow when `github.actor == 'renovate[bot]'`. `pull_request_target` runs the
   workflow file from the default branch; the PR's code is never checked out or run.
3. **Review job** (reusable workflow):
   1. Skips when the PR already carries `human-review-required`.
   2. Fetches an agent token from the broker (`AGENT_TOKEN_BROKER_ACTIONS_URL`), as in
      token-broker `docs/actions.md`.
   3. Checks out the base branch, so Claude can read the repository's `AGENTS.md`/`CLAUDE.md`
      for its dependency policy.
   4. Runs `claude-code-action` in automation mode with the federation inputs, the prompt file,
      `allowed_bots: renovate[bot]`, `--model claude-opus-5-5` and a JSON schema. Claude's tools
      are read-only: `gh pr view`, `gh pr diff`, `gh pr checks`, `WebFetch`, `Read`, `Grep`,
      `Glob`. It returns `{"decision": "approve" | "human", "summary": "…", "concerns": ["…"]}`.
   5. Acts on the decision with the agent token, in workflow steps rather than Claude tools:
      `approve` submits an approving review with the summary; `human` adds the label and a
      comment listing the concerns.
   6. On any failure (token exchange, Claude run, malformed output) adds the label with the
      default `GITHUB_TOKEN`, so a broken review fails towards a person.
4. **Maintainer approves a labelled PR.** Renovate's auto-merge merges it. A second job in the
   reusable workflow, called by the caller on `pull_request_review` (`submitted`, state
   `approved`, reviewer not a bot), removes the label with `GITHUB_TOKEN`. It needs neither the
   broker nor Claude.
5. **Renovate rebases.** The default ruleset dismisses stale reviews on push, so `synchronize`
   reviews again.

Concurrency: one review per PR, superseded runs cancelled (`renovate-review-<pr>`).

### What Claude decides

The prompt asks for `human` when any of these hold, `approve` otherwise:

- a major version, a toolchain, or a base image of the shipped artifact;
- release notes or changelog mention a breaking change, a removed or renamed API, a changed
  default, a migration, or a licence change;
- the diff touches more than dependency manifests, lock files and pins (e.g. generated code,
  source files, workflow logic beyond a `uses:` pin);
- a required check failed;
- the repository's `AGENTS.md` or `CLAUDE.md` names the dependency as hand-reviewed;
- Claude cannot find release notes or is unsure.

Text fetched from release notes is data, not instructions; the prompt says so, and Claude has
no tool that can approve, label or merge, so injected text can only change the JSON it returns.

### Identity and authorization

**Federation rule** (Claude Console, `workspace:inference`, 600 s), one per repository:

```json
"match": {
  "subject_prefix": "repo:pflege-de-labs/<repo>:pull_request",
  "audience": "https://api.anthropic.com",
  "claims": {
    "repository_owner": "pflege-de-labs",
    "event_name": "pull_request_target",
    "actor": "renovate[bot]",
    "base_ref": "main"
  }
}
```

- `event_name: pull_request_target` limits tokens to workflow files on the default branch, which
  need a code-owner review to change.
- `actor: renovate[bot]` cannot be taken by a user. A person pushing to a Renovate branch or
  reopening its PR gets no token; the failure step labels the PR for a person.
- `job_workflow_ref` is not matched: callers pin the reusable workflow by sha, so it changes with
  every bump, and the Console matches claims exactly.
- GitHub does not document the `sub` of a `pull_request_target` run. The first run's entry in
  the Console's authentication history shows it; if it is `…:ref:refs/heads/main`, use that
  prefix instead.

**Broker:** no change. `allowedRefs` already admits `refs/heads/main`. `allowedWorkflows` could
pin `pflege-de-labs/github-workflows/.github/workflows/renovate-review.yml@*`, but it is
deployment-wide and would cut off every other agent workflow; left empty.

**Variables:** `AGENT_TOKEN_BROKER_ACTIONS_URL` and `AGENT_RUNNER` (org, existing),
`ANTHROPIC_ORGANIZATION_ID` (org), `ANTHROPIC_RENOVATE_REVIEW_RULE_ID` (repository). The reusable
workflow reads them from the caller's `vars`; it has no inputs.

### Caller workflow

```yaml
name: Renovate review
on:
  pull_request_target:
    types: [opened, reopened, synchronize]
  pull_request_review:
    types: [submitted]
jobs:
  review:
    if: github.event_name == 'pull_request_target' && github.actor == 'renovate[bot]'
    permissions: { contents: read, pull-requests: write, id-token: write }
    uses: pflege-de-labs/github-workflows/.github/workflows/renovate-review.yml@<sha> # v1.2.0
  approved:
    if: >-
      github.event_name == 'pull_request_review' &&
      github.event.review.state == 'approved' &&
      github.event.pull_request.user.login == 'renovate[bot]'
    permissions: { pull-requests: write }
    uses: pflege-de-labs/github-workflows/.github/workflows/renovate-review.yml@<sha> # v1.2.0
```

The reusable workflow branches on `github.event_name`, so one file serves both jobs.

## Out of scope

- Renovate PRs into `pages` and `pages-v*` (teamster): a `pull_request_target` workflow runs from
  the base branch, which carries no workflows. They stay with a person.
- Claude pushing fixes to a Renovate branch: Renovate would stop managing it.
- github-workflows' own Renovate PRs: nothing merges itself there by design.

## Verification

- `actionlint` and the repository's own CI on the reusable workflow.
- In teamster, before the Monday run: trigger Renovate from the dependency dashboard. Check the
  Console's authentication history for the `sub`, then one PR each for approve (a digest bump),
  human by preset (a major) and human by Claude (a forced failing check).
- Approve a labelled PR as a maintainer and check that it merges and loses the label.

## Open

- The Renovate app's bypass on both teamster `main` rulesets (26037339, 22550878) is assumed
  from ADR 0004 and github-ops #199/#210; the rulesets' bypass lists are not readable with the
  agent token.
