# Axel-Automotive Organization Automation

This repository is the central hub for all organization-wide automation, shared workflows, and engineering standards across the Axel-Automotive GitHub organization. Any process that applies to multiple repositories lives here rather than being duplicated across individual repos.

Currently, this repository manages the automated release pipeline. As the organization grows, it will also house CI/CD workflows (linting, testing, building, deployment), shared action wrappers, repo provisioning scripts, and any other cross-cutting automation.


## What Lives Here

```
Axel-Automotive/.github/
    .github/
        workflows/
            release-governance.yml      # Central reusable release workflow
    workflow-templates/
        org-release.yml                 # Template for new repos
        org-release.properties.json     # Template metadata
    README.md                           # This file
```

### Planned Additions

As the organization's automation needs expand, this repository will grow to include additional reusable workflows for CI/CD stages (lint, test, build, deploy), shared scripts and tooling, organization-level issue and PR templates, and security and compliance workflows. The architecture is designed for this. Individual repos call into reusable workflows here, so adding a new org-wide process means adding a workflow to this repo and a thin caller to each consumer repo.


## Current Automation

### Automated Release Pipeline

Every repository enrolled in the release pipeline gets automatic versioning, changelog generation, AI-generated technical documentation, and GitHub Release publishing on every push to `main`. No manual tagging, versioning, or documentation updates are required from developers.

#### How It Works

Every enrolled repo has a thin caller workflow at `.github/workflows/org-release.yml`. This file contains no logic. It triggers on PRs and pushes to `main` and calls the central `release-governance.yml` workflow in this repository. All release logic executes in the context of the calling repo.

**On pull request to main:**

The workflow reads the latest git tag, determines the version bump type from PR labels, calculates the proposed next version, and posts a comment on the PR showing a preview of the version and release notes. If the PR is updated, the comment updates in place.

**On push to main (after merge or direct push):**

The workflow reads the latest git tag, determines the bump type, calculates the next version, generates release notes grouped by commit type, creates or updates `Documentation/CHANGELOG.md`, sends the repo's structure and recent changes to the Claude API to generate or update `Documentation/OVERVIEW.md`, creates an annotated git tag, and publishes a GitHub Release. All commits made by the workflow include `[skip ci]` to prevent infinite loops.

#### Versioning

The default strategy uses PR labels to determine the version bump.

| Label | Effect |
|---|---|
| `major` or `breaking` | Major bump (v1.0.0 to v2.0.0) |
| `minor`, `feature`, or `enhancement` | Minor bump (v1.0.0 to v1.1.0) |
| No matching label | Patch bump (v1.0.0 to v1.0.1) |

An alternative conventional commits strategy is available by changing `versioning_strategy` to `"conventional-commits"` in the caller workflow. With this strategy, `feat:` triggers a minor bump, any type followed by `!:` triggers a major bump, and everything else triggers a patch bump.

#### AI-Generated Documentation

On every release, the workflow gathers the repo's directory tree (up to 4 levels deep, capped at 200 entries), the first 100 lines of key configuration files (README.md, package.json, Cargo.toml, pyproject.toml, Dockerfile, docker-compose.yml, and others), a diff summary and excerpt since the last release, and the existing OVERVIEW.md if present. This context is sent to the Anthropic Claude API with instructions to produce or update a technical overview of the software. Claude updates existing documentation rather than rewriting from scratch, preserving accurate content and revising sections affected by recent changes.

If the API call fails for any reason, the workflow logs a warning and continues. Documentation generation never blocks a release.

#### What Each Repo Gets

After every merge to main, each enrolled repo ends up with:

A new semantic version tag (e.g., `v1.2.3`).
An updated `Documentation/CHANGELOG.md` with grouped release notes prepended.
An updated `Documentation/OVERVIEW.md` with a current technical overview of the codebase.
A GitHub Release with the version number and release notes.


## Enrolled Repositories

| Repository | Status |
|---|---|
| Axel-Automotive/Xpert-QA | Active |
| Axel-Automotive/user_analytics_dashboard | Active |
| Axel-Automotive/website-axelautomotive | Active |
| Axel-Automotive/Axel-Automotive-MAS | Active |
| Axel-Automotive/Xpert-Approval-Module | Active |
| Axel-Automotive/Axel-UIUX-Standards | Active |
| Axel-Automotive/Storybook | Active |

The `.github` repository itself is excluded because it is infrastructure, not application code.


## Adding a New Repository

When a new repository is created in the organization, three steps are required.

**1. Add the caller workflow.** Copy `.github/workflows/org-release.yml` from any enrolled repo into the new repo at the same path. Alternatively, use the workflow template in the GitHub UI under Actions > New workflow > "By Axel-Automotive."

**2. Set the Anthropic API key.**
```
gh secret set ANTHROPIC_API_KEY --repo Axel-Automotive/NEW_REPO_NAME
```

**3. Set workflow permissions.**
```
gh api repos/Axel-Automotive/NEW_REPO_NAME/actions/permissions/workflow -X PUT -f default_workflow_permissions=write
```

If the organization upgrades to GitHub Team, steps 2 and 3 become unnecessary. The org-level secret and permissions propagate automatically.


## Secrets

**ANTHROPIC_API_KEY**: Used for AI-generated documentation. Must be set as a repo-level secret on each enrolled repository on the current free plan. On GitHub Team, this can be a single org-level secret with visibility set to all repos.

**github.token**: The built-in GitHub Actions token. Automatically available in every workflow run. Used for checking out code, pushing commits, creating tags, posting PR comments, and creating GitHub Releases. No configuration required.

**ORG_RELEASE_TOKEN**: A Personal Access Token stored as an org-level secret. Currently unused by the workflow (github.token handles all operations) but available for future cross-repo automation that may require elevated permissions.


## Organization Settings

The following settings are configured at the org level.

**Actions permissions** (https://github.com/organizations/Axel-Automotive/settings/actions): Set to "Allow Axel-Automotive, and select non-Axel-Automotive, actions and reusable workflows." GitHub-authored actions (`actions/*`) are allowed. All other third-party actions are blocked. This prevents individual repos from pulling in unvetted actions.

**Organization ruleset** (https://github.com/organizations/Axel-Automotive/settings/rules): "Protect main branches" is defined, targeting all repositories and the default branch. It requires PRs before merging with 1 approval, requires the `release-governance / pr-validation` status check, restricts deletions, and blocks force pushes. This ruleset is not enforced on the free plan. It activates automatically upon upgrading to GitHub Team.

**Workflow permissions**: Set to read-write at the org level. Individual enrolled repos also have workflow permissions set to read-write to allow the release workflow to push commits, create tags, and post comments.


## Limitations

### GitHub Free Plan

Organization rulesets are defined but not enforced. Developers can push directly to main, bypassing PR validation. Org-level secrets do not inherit into reusable workflows, requiring per-repo secret setup. Required workflows cannot be enforced at the org level, so enrollment is manual.

### Workflow Constraints

The caller workflow must exist in each repo because GitHub requires a workflow file to physically exist inside a repository for it to trigger on that repo's events. Version calculation is tag-based; manually creating or deleting tags outside the workflow can break the version sequence. The AI documentation is generated from directory structure, config files, and a capped diff excerpt (500 lines), so very large releases may not be fully reflected. The changelog quality depends on commit message quality. Direct pushes to main skip PR validation and default to a patch bump.

### Operational Constraints

No build, test, lint, or deploy stages exist yet. No rollback mechanism is provided; bad releases require manual tag deletion and commit reversion. No multi-environment support; every merge to main is treated as a release. No monorepo support; each repo is versioned as a single unit.


## Maintenance

### Changing the Release Process

All release logic lives in `.github/workflows/release-governance.yml` in this repository. Changes to this file take effect immediately across all enrolled repos because caller workflows reference `@main`.

```
cd ~/.github
# Edit .github/workflows/release-governance.yml
git add .github/workflows/release-governance.yml
git commit -m "description of change"
git push origin main
```

No changes to individual repos are needed.

### Pinning the Workflow Version

If stability is needed during a period of heavy changes to the pipeline, change `@main` to a specific tag (e.g., `@release-v1.0.0`) in the caller workflow files across all repos. This requires updating every repo's caller file.

### Monitoring

Each repo's Actions tab shows workflow run history with detailed step logs. The "Generate Documentation/OVERVIEW.md" step will show a warning if the Claude API call fails. GitHub Releases on each repo provide a chronological record of all releases.


## Upgrading to GitHub Team

Upgrading enables three capabilities that close the gaps in the current setup.

**Enforced rulesets.** The "Protect main branches" ruleset activates, blocking direct pushes to main and requiring the release governance check to pass before any PR can merge.

**Org-level secret inheritance.** The ANTHROPIC_API_KEY can be set once at the org level and will be available in all repos without per-repo configuration.

**Required workflows.** The org can require the release governance workflow to run before any merge, eliminating the possibility of a repo opting out.

These changes eliminate all manual per-repo setup for new repositories and close every enforcement gap.


## Roadmap

The following automation is planned for this repository. Each addition follows the same pattern: add a reusable workflow here, add a thin caller to enrolled repos.

**Code linting.** A reusable workflow that detects each repo's language and runs the appropriate linter (ESLint, Stylelint, Ruff, etc.) on PRs.

**Automated testing.** A reusable workflow that runs test suites (npm test, pytest, etc.) if defined in the repo's configuration.

**Build verification.** A reusable workflow that builds the application to confirm it compiles and packages correctly.

**Deployment.** A reusable workflow that deploys to staging or production after successful builds and tests.

**Slack notifications.** A step in the release workflow that posts release announcements to a team Slack channel.

**Repo provisioning.** A script or GitHub App that automatically adds caller workflows, sets secrets, and configures permissions when new repos are created.

**Merge queue support.** Enable the `merge_group` trigger in the caller workflows and the org ruleset for repos with high merge volume.
