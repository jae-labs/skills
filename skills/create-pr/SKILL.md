---
name: create-pr
description: Inspects local git changes and generates a structured, high-quality PR description using the gh CLI. You MUST use this skill whenever the user asks to create a pull request, create a github pr, run '/create-pr', draft a PR description, summarize branch changes, review a diff before committing, or write a squash commit message. Even if the user only asks to "write a PR summary," invoke this skill to ensure proper branch hygiene and formatting.
---

# Pull Request Creator & Description Generator

Analyzes git changes between the current branch and the default branch to verify branch hygiene, inspect code modifications, and generate a clear, focused pull request description. Finally, automates PR creation using the GitHub CLI (`gh`).

## Core Principles

- **Inspection before creation**: Always assess the quality, size, and focus of the changes before writing any summary.
- **Documentation before creation**: Refresh `README.md`, `AGENTS.md`, and `CLAUDE.md` with `md-docs` before writing the PR if those files are stale or should reflect the final reviewed state of the branch.
- **Conciseness and clarity**: Provide high-level context to human reviewers rather than listing every minor file change.
- **Strict separation of concern**: Ensure PRs are focused on a single logical change. If unrelated modifications exist, alert the user.
- **Accurate signaling**: Highlight architectural decisions, security/performance considerations, and explicitly document all API modifications or breaking changes.

## Workflow

1. **Verify Branch State**:
   - Check if you are currently on the default/main branch (e.g., `main` or `master`).
   - If checked out on the default branch, you MUST switch to or create a new feature branch (e.g. `git checkout -b <feature-branch-name>`) before proceeding. Do not build a PR directly off the default branch.
   - Run `git status --porcelain` to check for uncommitted modifications. If any exist, warn the user and recommend committing them before generating the PR.
2. **Refresh Documentation**:
   - Before drafting the PR, verify whether `README.md`, `AGENTS.md`, or `CLAUDE.md` should change based on the final reviewed branch state.
   - If yes, run `md-docs` first and complete those documentation updates before generating the PR summary.
3. **Check for GitHub CLI**: Verify that the `gh` command-line tool is installed and authenticated.
4. **Determine the Default Branch**: Locate the target default branch (usually `main` or `master`) via remote configuration (`git remote show origin`) or branch listing.
5. **Inspect the Diff & Commits**:
   - Get the list of commits and their messages from the current branch relative to the default branch (e.g., `git log <default-branch>..HEAD`).
   - Compare the current branch against the default branch: `git diff <default-branch>...HEAD`
   - Inspect the commit messages, paths affected, file changes, and overall size of the diff.
6. **Evaluate Branch Hygiene**:
   - Verify if the changes represent a single focused logical change.
   - If the PR contains unrelated concerns (e.g. mixed features, multiple distinct bugfixes, major format changes), alert the user and recommend splitting the branch.
   - Ensure changes make the code easy to review without breaking runtime behavior.
7. **Analyze for Breaking Changes**: Look for API modifications, database schema changes, configuration updates, or dependency migrations.
8. **Generate PR Title and Description**: Construct the title and description using the structure defined below.
9. **Await User Confirmation**: Present the proposed PR Title and PR Description Body. You MUST explicitly ask the user for approval. **NEVER run `gh pr create` without the user's explicit confirmation.**
10. **Create the Pull Request**: Once approved, run the `gh pr create` CLI command using the approved title and body.

## Output Structure

Provide two separate outputs when presenting the generated PR description to the user:
1. The **Title** as plain text (not in a markdown block).
2. The **PR Description Body** in a single markdown code block (do NOT nest markdown blocks).

### Body Template

The body must follow this structure exactly:

```markdown
[1-2 sentence overview paragraph explaining the purpose and motivation of the changes.]

## Description

- **What**: Clear explanation of the changes introduced.
- **Why**: Context on why these changes are necessary (include links to Linear tasks or issue trackers if applicable).
- **How**: Brief explanation of the technical approach taken.

## Changes

- **[Component/Area]**: High-level context of what changed, including why and how. Use repository-relative paths (e.g., `lib/auth/session.go`) to reference key files.
- **[Component/Area]**: High-level context of what changed.

## Key Decisions & Breaking Changes

[Deep-dive into key architectural decisions, trade-offs, and an explicit call-out of any API modifications or breaking modifications.]
```

## Guidelines

**Do:**
- Group file-level changes into cohesive logical areas/components.
- Use only repository-relative paths when referencing files or folders.
- Highlight important technical details (e.g., timeouts, capacity limits, network protocols).
- Focus on big-picture impact to help human reviewers read quickly.
- Explicitly call out database migrations, breaking changes, or major dependency updates.

**Don't:**
- List trivial changes (e.g., package import changes, formatting, minor typo fixes, or basic logging additions) unless they are the primary focus of the PR.
- Include the branch name in the PR title or description (redundant).
- Use absolute filesystem paths (e.g., `/Users/username/...`).
- Treat basic observability (like standard logging) as a feature.
- Nest markdown blocks inside the body block.

## Example

**Title:**
Migrate API service to multi-stage Docker build on AWS ECS Fargate via Terraform

**Body:**
```markdown
Migrates the core API service deployment from a single-stage Docker build to a multi-stage container running on AWS ECS Fargate, managed by Terraform.

## Description

- **What**: Optimized the container image size via multi-stage Docker builds and updated the deployment infrastructure on AWS.
- **Why**: The previous container image was 1.2GB, causing slow ECS task startups. We also need to move to Fargate to eliminate EC2 host management overhead. (Linear: DEVOPS-412)
- **How**: Created a multi-stage `Dockerfile` to produce a 45MB runtime image, and updated our Terraform modules to provision the Fargate task definition and service.

## Changes

- **Containerization**: Updated `docker/Dockerfile` to use a two-stage build (build on `golang:1.21-alpine`, run on `alpine:3.19`), dropping root privileges for the runtime process.
- **Infrastructure**: Modified `terraform/ecs_service.tf` to define the Fargate capacity provider, security groups, and updated environment variables.

## Key Decisions & Breaking Changes

- **Non-root container user**: The runtime container now runs as a non-root user (`appuser`). If any local file writing is needed in future tasks, it must write to `/tmp`.
- **Breaking Change**: The Terraform variable `ecs_instance_type` has been removed as Fargate manages resources using cpu/memory configuration at the task level.
```
