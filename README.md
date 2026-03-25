# merge-queues

Understanding how merge queues interact with CI.

## Overview

This repository demonstrates how to use [GitHub Merge Queues](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue) with required status checks at three distinct stages of the merge lifecycle.

## The three check stages

| Stage | Event trigger | Job name | When it runs |
|-------|--------------|----------|--------------|
| **(A) Push check** | `push` | `Push Check` | Every time commits are pushed to any branch |
| **(B) PR check** | `pull_request` | `PR Check` | When a pull request is opened or updated |
| **(C) Merge-queue check** | `merge_group` | `Merge Queue Check` | After a PR enters the merge queue, before it is merged |

All three jobs live in [`.github/workflows/required-checks.yml`](.github/workflows/required-checks.yml).

## How to configure branch protection

To enforce all three checks on your base branch (e.g. `main`):

1. Go to **Settings → Branches → Add branch protection rule** for `main`.
2. Enable **Require status checks to pass before merging**.
3. Search for and add the following required checks:
   - `Push Check` — blocks a PR from being enqueued until the push check on the head branch passes.
   - `PR Check` — blocks a PR from being enqueued until the PR check passes.
4. Enable **Require merge queue**.
5. In the merge queue settings, add `Merge Queue Check` as a required check — this blocks the actual merge if the check fails after the PR is enqueued.

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
