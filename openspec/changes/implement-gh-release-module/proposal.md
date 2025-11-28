# Change: Implement GitHub Squash-Merge-Release Dagger Module

## Why

Automating the squash-merge-release workflow currently requires local `gh` CLI installation and manual coordination of multiple git and GitHub operations. By containerizing this workflow in a Dagger module, we eliminate local tooling dependencies, ensure consistent execution across environments, and provide a single command to validate, squash, merge, and tag releases.

## What Changes

- **NEW** Python Dagger module (`dagger-gh-release`) following x5 API conventions
- **NEW** Container-based `gh` CLI execution using `ghcr.io/cli/cli:latest`
- **NEW** GitHub token authentication via `dagger.Secret` (supports both github.com and GitHub Enterprise)
- **NEW** Auto-detection of `GH_HOST` from git remote URL (SSH and HTTPS formats)
- **NEW** Pre-flight validation function to check branch state and commits ahead
- **NEW** Merge conflict detection with automatic halt and resolution instructions
- **NEW** Large commit count guardrail (>100 commits requires explicit `--force-large-squash` flag)
- **NEW** Main `squash-merge-release` function with required parameters (non-interactive)
- **NEW** VERSION file detection (root or `version/VERSION`) for automatic tagging
- **NEW** Git push authentication via HTTPS token injection
- **NEW** Structured error messages with actionable guidance and best practice warnings

## Impact

- Affected specs: `squash-merge-release` (new capability)
- Affected code:
  - `src/main/__init__.py` - Main module implementation
  - `pyproject.toml` - Python dependencies
  - `dagger.json` - Module configuration
  - `README.md` - User documentation

## Out of Scope

- SSH key authentication for git push (HTTPS only)
- Interactive prompts (Dagger functions are non-interactive)
- GitHub Actions integration
- Multi-remote support (only `origin`)
- Local git state changes after merge (user must sync manually)