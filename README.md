# GitHub Actions Corner Case Test

This repository is dedicated to testing a particular behavior of GitHub Actions, specifically how workflows behave when they are triggered by events in a different branch from where they were originally created.

## Problem Statement

What happens if we:

1. Create a branch called `test`.
2. Switch back to `main` and create a workflow that triggers on pushes to `test`.
3. Push an arbitrary commit to `test` (before the workflow exists in that branch).

## Expected vs. Actual Results

**Expectation:** The workflow should trigger because it was defined in `main` and explicitly listens for pushes to `test`.  

**Actual Result:** Nothing happens.  

This behavior is explained in the [GitHub documentation](https://docs.github.com/en/actions/writing-workflows/choosing-when-your-workflow-runs/triggering-a-workflow#about-workflow-triggers):

> The following steps occur to trigger a workflow run:
>
> 1. An event occurs on your repository. The event has an associated commit SHA and Git ref.
> 2. GitHub searches the `.github/workflows` directory in the root of your repository for workflow files that exist in the commit SHA or Git ref of the event.
> 3. A workflow run is triggered for any workflows with `on:` values that match the triggering event. Some events also require the workflow file to be present on the default branch of the repository to run.
> 4. Each workflow run uses the version of the workflow that exists in the commit SHA or Git ref of the event. When a workflow runs, GitHub sets the `GITHUB_SHA` (commit SHA) and `GITHUB_REF` (Git ref) environment variables in the runner.

The key insight lies in **step 2**:  

When a commit occurs, GitHub looks for workflows inside that commit’s `.github/workflows` directory. If no workflow matching the triggering event exists **within the commit itself**, nothing happens.

![image](https://github.com/user-attachments/assets/cb8abe76-5b3f-4ae8-8b11-4c1a1788cc36)

## Explanation with an Example

Consider the following sequence of events:

1. We branch off `test` from `main`.
2. In `main`, we add a new workflow that triggers on pushes to `test`.
3. We push a commit to `test`.

At this point, the latest commit in `test` does not contain the workflow (because it was added in `main`, not `test`). Since GitHub only searches for workflows **inside the commit that triggered the event**, no workflow is found in `test`, so nothing runs.

![image](https://github.com/user-attachments/assets/e1f21b49-ebf1-4ff4-bab7-f6313893bc8f)

## Why Is This Behavior Important?

This restriction exists primarily for **security reasons**. If GitHub allowed workflows defined in `main` to trigger from any branch:

- A **malicious actor** could fork your repository and create a workflow that executes unauthorized code when merged.
- A contributor could unknowingly introduce a workflow that runs on `main` and leaks repository secrets.
- Workflows could be added dynamically in PRs, running arbitrary scripts **before maintainers review them**.

By requiring workflows to be **present in the commit that triggers the event**, GitHub ensures that only reviewed and merged workflows can execute. This prevents unauthorized code execution and protects repository secrets from exposure.
