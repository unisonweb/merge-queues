# merge-queues

Understanding how merge queues interact with CI.

## Overview

This repository demonstrates how to use [GitHub Merge Queues](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue) with required status checks at three distinct stages of the merge lifecycle.

## The three check stages

| Stage | Event trigger | Job name | Passes when… |
|-------|--------------|----------|--------------|
| **(A) Push check** | `push` | `Push Check` | `push` is in the `checks` file |
| **(B) PR check** | `pull_request` | `PR Check` | `pr` is in the `checks` file |
| **(C) Merge-queue check** | `merge_group` | `Merge Queue Check` | `merge_queue` is in the `checks` file |

All three jobs live in [`.github/workflows/required-checks.yml`](.github/workflows/required-checks.yml).

### Controlling pass/fail via the `checks` file

The [`checks`](checks) file at the repository root contains a space-separated list of tokens.
Each check looks for its own token in that file:

| Token | Controls |
|-------|----------|
| `push` | Push Check (A) |
| `pr` | PR Check (B) |
| `merge_queue` | Merge Queue Check (C) |

To make a check **pass**: ensure its token is present in the file.  
To make a check **fail**: remove its token from the file and commit.

**Example — all checks pass:**
```
push pr merge_queue
```

**Example — only Push Check and PR Check pass (Merge Queue Check fails):**
```
push pr
```

## How to configure branch protection

To enforce all three checks on your base branch (e.g. `main`):

1. Go to **Settings → Branches → Add branch protection rule** for `main`.
2. Enable **Require status checks to pass before merging**.
3. Search for and add the following required checks:
   - `Push Check` — blocks a PR from being enqueued until the push check on the head branch passes.
   - `PR Check` — blocks a PR from being enqueued until the PR check passes.
4. Enable **Require merge queue**.
5. Still on the same branch-protection rule page, scroll to the **Merge queue** section and add `Merge Queue Check` as a required status check — this blocks the actual merge if the check fails after the PR is enqueued.

## How the merge queue flow works

```
developer pushes commits
        │
        ▼
  [Push Check] ← (A) required on push
        │  passes
        ▼
  developer opens / updates PR
        │
        ▼
  [PR Check] ← (B) required on pull_request
        │  passes
        ▼
  PR is added to the merge queue
        │
        ▼
  [Merge Queue Check] ← (C) required on merge_group
        │  passes
        ▼
  PR is merged into main
```

If the **Merge Queue Check** (C) fails after the PR has been enqueued, GitHub automatically removes the PR from the queue and the merge is blocked until the issue is resolved.

