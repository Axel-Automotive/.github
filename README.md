# Axel-Automotive Automated Release Pipeline

## What This Is

This is an organization-wide automated release pipeline that runs on every push to `main` across all repositories in the Axel-Automotive GitHub organization. It handles semantic versioning, git tagging, changelog generation, AI-generated technical documentation, and GitHub Release publishing. No manual versioning, tagging, or documentation updates are required from developers.

This is not a full CI/CD pipeline. It does not build, test, lint, or deploy code. It is the release governance and documentation layer. Build, test, and deploy stages can be added to the same centralized architecture in the future.


## Architecture

The system uses GitHub's reusable workflow feature to centralize all release logic in one repository while running it in the context of every other repository in the organization.

### The Three Components

**1. Central Reusable Workflow**
Location: `Axel-Automotive/.github/.github/workflows/release-governance.yml`

This is the single source of truth. It contains all versioning logic, changelog generation, AI documentation generation, tagging, and release publishing. Every repo calls into this file. When the release process needs to change, this is the only file that gets modified. No other repository needs to be touched.

**2. Thin Caller Workflow**
Location in each repo: `.github/workflows/org-release.yml`

A small configuration file that exists in every repository. It contains no logic. It declares when to trigger (on PR and push to main), sets permissions, and calls the central reusable workflow. This file exists because GitHub requires a workflow file to physically exist inside a repository for it to trigger on that repository's events. There is no way around this requirement.

**3. Org-Level Configuration**
The Axel-Automotive GitHub organization has the following configured:

Actions permissions: set to allow Axel-Automotive actions and GitHub-authored actions only. Third-party actions are blocked.

Organization ruleset: "Protect main branches" is defined but not enforced on the current free plan. It requires PRs before merging, requires the `release-governance / pr-validation` status check to pass, restricts branch deletions, and blocks force pushes. This ruleset activates automatically upon upgrading to GitHub Team.

Workflow permissions: set to read-write at both the org and repo level, allowing the workflow to push commits (changelog, documentation), create tags, and post PR comments.


## Repository Structure

### Org .github Repository

```
Axel-Automotive/.github/
    .github/
        workflows/
            release-governance.yml      # Central reusable workflow (all logic)
    workflow-templates/
        org-release.yml                 # Template for new repos
        org-release.properties.json     # Template metadata
```

### Every Other Repository

```
any-repo/
    .github/
        workflows/
            org-release.yml             # Thin caller (no logic)
    Documentation/
        CHANGELOG.md                    # Auto-generated, updated each release
        OVERVIEW.md                     # AI-generated, updated each release
```


## What Happens On Each Event

### When a Pull Request Is Opened or Updated Targeting Main

1. The caller workflow in the repo triggers.
2. The central reusable workflow's `pr-validation` job runs.
3. It checks out the repository with full git history.
4. It reads the latest git tag (or defaults to `v0.0.0` if none exists).
5. It determines the version bump type. If using the labels strategy (default), it reads the PR's labels. `major` or `breaking` labels trigger a major bump. `minor`, `feature`, or `enhancement` labels trigger a minor bump. No matching label defaults to a patch bump.
6. It calculates the proposed next semantic version.
7. It generates a preview of the release notes from commit messages since the last tag.
8. It posts a comment on the PR showing the proposed version, bump type, and release notes preview.
9. If the PR is updated, the comment is updated in place rather than posting a new one.

### When a Push to Main Occurs (After PR Merge or Direct Push)

1. The caller workflow in the repo triggers.
2. The central reusable workflow's `release` job runs.
3. It checks out the repository with full git history.
4. It reads the latest git tag.
5. It checks if there are any new commits since the last tag. If not, it skips the release entirely.
6. It determines the version bump type from the most recently merged PR's labels (or from commit messages if using conventional commits strategy).
7. It calculates the authoritative next semantic version.
8. It generates grouped release notes. Commits are categorized by conventional commit type (feat, fix, docs, chore, etc.) if the commit messages follow that format. Uncategorized commits go under "Other."
9. It creates or updates `Documentation/CHANGELOG.md` by prepending the new release notes to the existing file. If the file does not exist, it creates it.
10. It commits and pushes the changelog with the message `chore(release): update changelog for vX.Y.Z [skip ci]`.
11. It gathers the repository's directory structure, key configuration files (README.md, package.json, Cargo.toml, etc.), the diff since the last release, and the existing OVERVIEW.md if present.
12. It sends all of this context to the Anthropic Claude API (claude-sonnet-4-20250514) with instructions to generate or update a technical overview of the software.
13. It writes the response to `Documentation/OVERVIEW.md`, commits, and pushes with the message `docs(release): update OVERVIEW.md for vX.Y.Z [skip ci]`.
14. It pulls the latest state with rebase to handle any race conditions, then creates an annotated git tag with the version number.
15. It pushes the tag.
16. It creates a GitHub Release using the tag, with the release notes as the body.

The `[skip ci]` in the commit messages prevents the workflow from triggering itself in an infinite loop.


## Versioning Strategy

The pipeline supports two strategies, configured per-repo in the caller workflow's `versioning_strategy` input.

### Labels (Default)

Developers or reviewers add labels to PRs before merging to control the version bump.

| Label | Effect |
|---|---|
| `major` or `breaking` | Major bump (v1.0.0 to v2.0.0) |
| `minor`, `feature`, or `enhancement` | Minor bump (v1.0.0 to v1.1.0) |
| No matching label | Patch bump (v1.0.0 to v1.0.1) |

This is the default because it requires no changes to developer commit habits.

### Conventional Commits

The pipeline parses commit messages to determine the bump type.

`feat:` or `feat(scope):` triggers a minor bump.
`fix:`, `chore:`, `docs:`, and all other types trigger a patch bump.
Any type followed by `!:` (e.g., `feat!:` or `fix!:`) triggers a major bump.

This requires developers to follow the conventional commits specification in their commit messages.


## AI-Generated Documentation

On every release, the pipeline sends the following context to the Claude API:

The repository's directory tree (up to 4 levels deep, excluding hidden directories, node_modules, vendor, dist, and build directories, capped at 200 entries).

The first 100 lines of key configuration and entry files: README.md, package.json, Cargo.toml, pyproject.toml, setup.py, go.mod, composer.json, Gemfile, pom.xml, build.gradle, Makefile, docker-compose.yml, and Dockerfile.

A git diff stat summary showing which files changed since the last release (up to 50 lines).

A raw diff excerpt (up to 500 lines) showing the actual code changes.

The existing OVERVIEW.md content, if one already exists.

Claude is instructed to produce or update a technical overview covering what the software does, its architecture and project structure, key modules and their responsibilities, how the pieces connect, external dependencies, and how to run or build the project. When an existing OVERVIEW.md is provided, Claude updates it rather than rewriting from scratch, preserving accurate existing content and revising sections affected by the latest changes.

If the Claude API call fails for any reason (network error, authentication failure, rate limit), the workflow logs a warning and continues. The release is not blocked by a documentation failure.


## Secrets

The pipeline uses two secrets.

**ANTHROPIC_API_KEY**: The Anthropic API key used for generating OVERVIEW.md. On the current free GitHub plan, this secret must be set individually on each repository because org-level secrets do not inherit into reusable workflows on the free plan. This can be set via:

```
gh secret set ANTHROPIC_API_KEY --repo Axel-Automotive/REPO_NAME
```

**github.token**: The built-in GitHub Actions token. This is automatically available in every workflow run and does not need to be configured. It is used for checking out code, pushing commits, creating tags, posting PR comments, and creating GitHub Releases. It works because the caller workflow declares `permissions: contents: write` and `pull-requests: write`.

**ORG_RELEASE_TOKEN**: A Personal Access Token that was created as an org-level secret. It is currently unused by the workflow (the pipeline uses `github.token` instead) but remains available if future cross-repo operations require it.


## Enrolled Repositories

The following repositories currently have the release pipeline active:

1. Axel-Automotive/Xpert-QA
2. Axel-Automotive/user_analytics_dashboard
3. Axel-Automotive/website-axelautomotive
4. Axel-Automotive/Axel-Automotive-MAS
5. Axel-Automotive/Xpert-Approval-Module
6. Axel-Automotive/Axel-UIUX-Standards
7. Axel-Automotive/Storybook

The `.github` repository itself is excluded because it is infrastructure, not application code.


## Adding a New Repository

When a new repository is created in the Axel-Automotive organization, three things must be done manually:

**1. Add the caller workflow.**
Copy `.github/workflows/org-release.yml` from any enrolled repo into the new repo at the same path. Or use the workflow template available in the GitHub UI under Actions > New workflow > "By Axel-Automotive."

**2. Set the Anthropic API key.**

```
gh secret set ANTHROPIC_API_KEY --repo Axel-Automotive/NEW_REPO_NAME
```

**3. Set workflow permissions.**

```
gh api repos/Axel-Automotive/NEW_REPO_NAME/actions/permissions/workflow -X PUT -f default_workflow_permissions=write
```

If the organization upgrades to GitHub Team, steps 2 and 3 become unnecessary because the org-level secret and permissions propagate automatically.


## Limitations

### GitHub Free Plan Constraints

**Organization rulesets are not enforced.** The "Protect main branches" ruleset exists but has no effect until the organization upgrades to GitHub Team or Enterprise. This means developers can push directly to main, bypassing the PR validation step. The release job still runs on direct pushes, but there is no gate preventing unreviewed code from reaching main.

**Org-level secrets do not inherit into reusable workflows.** The `ANTHROPIC_API_KEY` must be set as a repo-level secret on each individual repository. The `ORG_RELEASE_TOKEN` org secret was set but is not currently used because `github.token` handles all operations the workflow needs.

**Required workflows cannot be enforced at the org level.** On the free plan, there is no mechanism to force every repo to have the caller workflow file. Enrollment is manual.

### Workflow Limitations

**The caller workflow must exist in each repo.** GitHub does not support triggering a workflow from one repo based on events in another repo. Each repo needs the thin caller file physically committed.

**Version calculation is tag-based.** If someone manually creates or deletes tags outside the workflow, the version sequence can become inconsistent. Tags should only be created by the workflow.

**The AI documentation is best-effort.** The OVERVIEW.md is generated from directory structure, config files, and diffs. It does not read every source file in the repo. For large or complex codebases, the overview will cover high-level architecture accurately but may miss implementation details buried in files the prompt did not include.

**The diff excerpt sent to Claude is capped at 500 lines.** For very large releases with many changes, Claude will only see a subset of the diff. The overview will reflect the changes it can see.

**The changelog is generated from commit messages.** If commit messages are vague ("fix stuff", "update"), the changelog will be vague. The quality of the changelog directly reflects the quality of commit messages.

**Direct pushes to main skip PR validation.** The version preview comment and label-based bump detection only work on PRs. A direct push to main will still trigger the release job, but the bump type will default to patch unless conventional commits are detected in the commit messages.

### Operational Limitations

**No build, test, lint, or deploy steps.** This pipeline handles release governance only. Adding CI/CD stages requires extending the reusable workflow or adding additional reusable workflows.

**No rollback mechanism.** If a bad release is published, you must manually delete the tag and GitHub Release, then revert the commit. The pipeline does not provide automated rollback.

**No multi-environment support.** The pipeline does not distinguish between staging and production. Every merge to main is treated as a production release.

**No monorepo support.** The pipeline treats each repository as a single releasable unit. If a repository contains multiple independently versioned packages, the pipeline will version them as one.


## Maintenance

All release logic lives in one file: `Axel-Automotive/.github/.github/workflows/release-governance.yml`. Changes to this file take effect immediately across all enrolled repositories because the caller workflows reference `@main`.

The caller workflow files in individual repos should never be modified unless the org name changes or the workflow inputs need to be overridden for a specific repo.

If you need to pin the reusable workflow to a specific version for stability (e.g., during a period of heavy changes to the pipeline), change `@main` to a tag like `@release-v1.0.0` in the caller workflow files across all repos.

### Updating the Reusable Workflow

```
cd ~/.github
# Make changes to .github/workflows/release-governance.yml
git add .github/workflows/release-governance.yml
git commit -m "description of change"
git push origin main
```

Every repo picks up the change on its next workflow run.

### Monitoring

Each repo's Actions tab shows the history of all workflow runs. Failed runs show error details in the step logs. The "Generate Documentation/OVERVIEW.md" step will show a warning if the Claude API call fails, but the rest of the release process completes normally.

GitHub Releases on each repo provide a chronological record of every release with version numbers and release notes.


## Upgrading to GitHub Team

Upgrading the Axel-Automotive organization to GitHub Team enables:

**Enforced rulesets.** The "Protect main branches" ruleset activates, blocking direct pushes to main and requiring the `release-governance / pr-validation` check to pass before any PR can merge.

**Org-level secret inheritance.** The `ANTHROPIC_API_KEY` can be set once at the org level with `--visibility all` and will be available in all repos without per-repo configuration.

**Required workflows.** The org can require the release governance workflow to run before any merge to main, eliminating the possibility of a repo opting out by deleting the caller file.

These three changes eliminate all manual per-repo setup steps for new repositories and close the enforcement gaps that exist on the free plan.


## Future Additions

The centralized architecture supports adding the following without changing any individual repo:

**Linting.** Add a job to the reusable workflow that detects the repo's language and runs the appropriate linter (ESLint, Stylelint, Ruff, etc.).

**Testing.** Add a job that runs `npm test`, `pytest`, or whatever test runner the repo uses, if a test script is defined.

**Build verification.** Add a job that builds the application to confirm it compiles and packages correctly.

**Deployment.** Add a job that deploys to staging or production after a successful release.

**Slack notifications.** Add a step that posts release announcements to a Slack channel.

**Merge queue support.** Uncomment the `merge_group` trigger in the caller workflow and enable merge queue in the org ruleset.

All of these would be added to the central reusable workflow and would take effect across all enrolled repos with no changes to individual repositories.
