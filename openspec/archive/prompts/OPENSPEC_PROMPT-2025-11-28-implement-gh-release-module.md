# OpenSpec change prompt

## Context

Create a new Dagger module (`dagger-gh-release`) that implements the GitHub squash-merge-release workflow. This module containerizes the `gh` CLI to eliminate local `gh` installation requirements while supporting GitHub Enterprise via token-based authentication passed as Dagger Secrets.

Reference workflow: `dot_kilocode/workflows/gh-squash-merge-release.md`

## Goal

Build a Python Dagger module that performs: validate → squash commits → create PR → merge PR → create version tag. The module should work with any GitHub instance (github.com or enterprise) by accepting tokens as Secrets and auto-detecting the host from the git remote.

## Scope

### In scope
- Python Dagger module following x5 API conventions
- Use official `ghcr.io/cli/cli:latest` container image
- GitHub token passed as `dagger.Secret` (works with `--github-token=env:GITHUB_TOKEN`)
- Auto-detect `GH_HOST` from git remote URL (SSH and HTTPS formats)
- Pre-flight validation function (check branch, commits ahead, etc.)
- Main `squash-merge-release` function with required parameters (no interactive prompts)
- Support for VERSION file detection (root or `version/VERSION`)
- Git push authentication via HTTPS token injection
- Proper error messages with actionable guidance

### Out of scope
- SSH key authentication for git push (HTTPS only for now)
- Interactive prompts (Dagger functions are non-interactive)
- GitHub Actions integration
- Multi-remote support (only `origin`)
- Local git state changes (user must run `git checkout main && git pull` after)

## Desired behavior

- User runs: `dagger call -m gh-release squash-merge-release --source=. --github-token=env:GH_TOKEN --squash-message="feat: My changes"`
- Module validates: not on main/master, has commits ahead, working directory is clean
- Module squashes commits, force pushes, creates PR, merges with squash, creates version tag
- Returns success message with PR number, tag URL, and cleanup instructions

## Constraints & assumptions

- Assume `--source=.` always required (per Dagger module rules, no default directory)
- Assume user extracts token from `~/.config/gh/hosts.yaml` before calling (module doesn't read hosts.yaml directly)
- Assume HTTPS remotes for git push (inject token into git-credentials)
- Assume single `origin` remote
- Assume base branch is `main` (with `master` fallback)
- Git config for commits: `user.email=dagger@localhost`, `user.name=Dagger Workflow`

## Acceptance criteria

- [ ] Module builds and passes `dagger functions` check
- [ ] `squash-merge-release` function accepts: `source` (Directory), `github_token` (Secret), `squash_message` (str), `gh_host` (optional str with auto-detect)
- [ ] Works with github.com when `GH_HOST=github.com` and correct token
- [ ] Works with GitHub Enterprise when `GH_HOST=github.mycompany.com` and correct token
- [ ] Fails with clear error if: on main branch, no commits ahead, or token invalid
- [ ] Creates and merges PR successfully on test repository
- [ ] Creates version tag if VERSION file present
- [ ] Returns structured success message with PR URL and tag URL