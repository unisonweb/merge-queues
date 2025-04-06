# merge-queues

Understanding how merge queues interact with CI.

## Overview

This repository demonstrates how to use [GitHub Merge Queues](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue) with required status checks at three distinct stages of the merge lifecycle.

## The three check stages

| Stage | Event trigger | Job name | Passes when… |
|-------|--------------|----------|--------------|
| **(A) Push check** | `push` | `Push Check` | bit 0 of the last SHA nibble is **0** |
| **(B) PR check** | `pull_request` | `PR Check` | bit 1 of the last SHA nibble is **0** |
| **(C) Merge-queue check** | `merge_group` | `Merge Queue Check` | bit 2 of the last SHA nibble is **0** |

All three jobs live in [`.github/workflows/required-checks.yml`](.github/workflows/required-checks.yml).

Each check inspects a different bit of the last hexadecimal character of the commit SHA (`github.sha`), giving each check an independent ~50% chance of passing or failing on any given commit.  This lets you easily simulate mixed pass/fail outcomes across the three stages without needing to touch any source code.

### Pass/fail lookup table

The last nibble of a SHA can be `0`–`f`.  The table below shows which checks pass (`✓`) or fail (`✗`) for each value:

| Last nibble | bit 2 | bit 1 | bit 0 | Merge Queue Check | PR Check | Push Check |
|-------------|-------|-------|-------|:-----------------:|:--------:|:----------:|
| `0` | 0 | 0 | 0 | ✓ | ✓ | ✓ |
| `1` | 0 | 0 | 1 | ✓ | ✓ | ✗ |
| `2` | 0 | 1 | 0 | ✓ | ✗ | ✓ |
| `3` | 0 | 1 | 1 | ✓ | ✗ | ✗ |
| `4` | 1 | 0 | 0 | ✗ | ✓ | ✓ |
| `5` | 1 | 0 | 1 | ✗ | ✓ | ✗ |
| `6` | 1 | 1 | 0 | ✗ | ✗ | ✓ |
| `7` | 1 | 1 | 1 | ✗ | ✗ | ✗ |
| `8` | 0 | 0 | 0 | ✓ | ✓ | ✓ |
| `9` | 0 | 0 | 1 | ✓ | ✓ | ✗ |
| `a` | 0 | 1 | 0 | ✓ | ✗ | ✓ |
| `b` | 0 | 1 | 1 | ✓ | ✗ | ✗ |
| `c` | 1 | 0 | 0 | ✗ | ✓ | ✓ |
| `d` | 1 | 0 | 1 | ✗ | ✓ | ✗ |
| `e` | 1 | 1 | 0 | ✗ | ✗ | ✓ |
| `f` | 1 | 1 | 1 | ✗ | ✗ | ✗ |

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

