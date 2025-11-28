## ADDED Requirements

### Requirement: GitHub Host Auto-Detection

The module SHALL auto-detect the GitHub host from the git remote URL.

#### Scenario: SSH remote format detection
- **WHEN** the origin remote URL is in SSH format (`git@host:org/repo.git`)
- **THEN** the module SHALL extract and use the host portion (e.g., `github.com`, `github.mycompany.com`)

#### Scenario: HTTPS remote format detection
- **WHEN** the origin remote URL is in HTTPS format (`https://host/org/repo.git`)
- **THEN** the module SHALL extract and use the host portion

#### Scenario: Manual host override
- **WHEN** the user provides an explicit `--gh-host` parameter
- **THEN** the module SHALL use the provided host instead of auto-detection

#### Scenario: Fallback to github.com
- **WHEN** the remote URL format cannot be parsed
- **THEN** the module SHALL default to `github.com`

---

### Requirement: Pre-flight Validation

The module SHALL validate the repository state before performing any operations.

#### Scenario: Working directory must be clean
- **WHEN** the source directory contains uncommitted changes
- **THEN** the module SHALL return an error with the list of dirty files and instructions to commit or stash

#### Scenario: Not on protected branch
- **WHEN** the current branch is `main` or `master`
- **THEN** the module SHALL return an error indicating the workflow requires a feature branch

#### Scenario: Commits ahead of base
- **WHEN** the feature branch has no commits ahead of main/master
- **THEN** the module SHALL return an error indicating nothing to merge

#### Scenario: GitHub token validation
- **WHEN** the provided GitHub token is invalid or lacks required permissions
- **THEN** the module SHALL return an error with guidance on required token scopes

#### Scenario: Successful validation
- **WHEN** all pre-flight checks pass
- **THEN** the module SHALL proceed with the workflow OR return a validation success message (for the validate function)

---

### Requirement: Merge Conflict Detection

The module SHALL detect and halt on merge conflicts before squashing.

#### Scenario: Merge conflicts detected
- **WHEN** the feature branch cannot be cleanly rebased onto main/master
- **THEN** the module SHALL return an error indicating merge conflicts exist
- **AND** the module SHALL NOT proceed with any squash or push operations
- **AND** the error message SHALL instruct the user to resolve conflicts manually with `git rebase main`

#### Scenario: No merge conflicts
- **WHEN** the feature branch can be cleanly merged with main/master
- **THEN** the module SHALL proceed with the workflow

---

### Requirement: Large Commit Count Guardrail

The module SHALL warn and require explicit confirmation when squashing a large number of commits.

#### Scenario: Large commit count without confirmation
- **WHEN** the branch has more than 100 commits ahead of main
- **AND** the `force_large_squash` parameter is not set to `true`
- **THEN** the module SHALL return an error with:
  - The commit count
  - A warning that branches should not diverge this far from main
  - A suggestion to break up the work into smaller PRs
  - Instructions to pass `--force-large-squash=true` to proceed anyway

#### Scenario: Large commit count with confirmation
- **WHEN** the branch has more than 100 commits ahead of main
- **AND** the `force_large_squash` parameter is set to `true`
- **THEN** the module SHALL proceed with the workflow
- **AND** the result message SHALL include a caution about keeping branches closer to main

#### Scenario: Normal commit count
- **WHEN** the branch has 100 or fewer commits ahead of main
- **THEN** the module SHALL proceed without requiring additional confirmation

---

### Requirement: Commit Squashing

The module SHALL squash all commits on the feature branch into a single commit.

#### Scenario: Multiple commits squashed
- **WHEN** the branch has multiple commits ahead of main
- **THEN** the module SHALL reset to main and create a single commit with the provided squash message

#### Scenario: Single commit preserved
- **WHEN** the branch has only one commit ahead of main
- **THEN** the module SHALL use the existing commit (no squash needed) unless a different message is provided

#### Scenario: Force push after squash
- **WHEN** commits are squashed successfully
- **THEN** the module SHALL force push to the feature branch using `--force-with-lease`

---

### Requirement: Pull Request Creation

The module SHALL create a pull request from the feature branch to the base branch.

#### Scenario: PR with squash message as title
- **WHEN** creating a pull request
- **THEN** the module SHALL use the squash message as the PR title

#### Scenario: PR body contains commit history
- **WHEN** creating a pull request
- **THEN** the module SHALL include the original commit messages in the PR body

#### Scenario: Existing PR handling
- **WHEN** a PR already exists for the feature branch
- **THEN** the module SHALL use the existing PR instead of creating a new one

---

### Requirement: Pull Request Merge

The module SHALL merge the pull request using squash merge strategy.

#### Scenario: Squash merge with branch deletion
- **WHEN** merging the pull request
- **THEN** the module SHALL use `--squash --delete-branch` options

#### Scenario: Merge to main branch
- **WHEN** merging the pull request
- **THEN** the module SHALL merge to `main` (with fallback to `master` if main doesn't exist)

---

### Requirement: Version Tagging

The module SHALL create a version tag if a VERSION file exists.

#### Scenario: VERSION file at root
- **WHEN** a `VERSION` file exists at the repository root
- **THEN** the module SHALL read the version and create a tag `v{VERSION}`

#### Scenario: VERSION file in version directory
- **WHEN** a `version/VERSION` file exists
- **THEN** the module SHALL read the version and create a tag `v{VERSION}`

#### Scenario: No VERSION file
- **WHEN** no VERSION file is found
- **THEN** the module SHALL skip tagging without error

#### Scenario: Tag already exists
- **WHEN** the tag `v{VERSION}` already exists
- **THEN** the module SHALL return an error indicating the tag exists and suggest incrementing the version

#### Scenario: Annotated tag creation
- **WHEN** creating a version tag
- **THEN** the module SHALL create an annotated tag with message `Release {VERSION}`

---

### Requirement: Result Reporting

The module SHALL return a structured success message on completion.

#### Scenario: Success with version tag
- **WHEN** the workflow completes successfully with a version tag
- **THEN** the module SHALL return:
  - Git host used
  - PR number and URL
  - Tag name and URL
  - Cleanup instructions for local sync

#### Scenario: Success without version tag
- **WHEN** the workflow completes successfully without a VERSION file
- **THEN** the module SHALL return:
  - Git host used
  - PR number and URL
  - Note that no tag was created
  - Cleanup instructions for local sync

#### Scenario: Local cleanup instructions
- **WHEN** the workflow completes successfully
- **THEN** the module SHALL instruct the user to run `git checkout main && git pull` locally

---

### Requirement: Git Authentication for Push

The module SHALL authenticate git push operations using the provided GitHub token.

#### Scenario: HTTPS token injection
- **WHEN** pushing to the remote repository
- **THEN** the module SHALL use the GitHub token for HTTPS authentication via git credential helper

#### Scenario: Git identity configuration
- **WHEN** creating commits in the container
- **THEN** the module SHALL configure git with `user.email=dagger@localhost` and `user.name=Dagger Workflow`

---

### Requirement: Function Interface

The module SHALL expose public functions following Dagger x5 API conventions.

#### Scenario: squash_merge_release function signature
- **WHEN** calling the main workflow function
- **THEN** the function SHALL accept:
  - `source: Directory` (required) - Source directory containing git repository
  - `github_token: Secret` (required) - GitHub token for authentication
  - `squash_message: str` (required) - Commit message for the squashed commit
  - `gh_host: str` (optional) - Override auto-detected GitHub host
  - `force_large_squash: bool` (optional, default false) - Bypass the >100 commits guardrail

#### Scenario: validate function signature
- **WHEN** calling the validation-only function
- **THEN** the function SHALL accept:
  - `source: Directory` (required) - Source directory containing git repository
  - `github_token: Secret` (required) - GitHub token for authentication
  - `gh_host: str` (optional) - Override auto-detected GitHub host

#### Scenario: get_version function signature
- **WHEN** calling the version detection function
- **THEN** the function SHALL accept:
  - `source: Directory` (required) - Source directory to check for VERSION file
- **AND** return the version string or empty string if not found