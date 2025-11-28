## Context

This module provides a containerized workflow for the common "squash commits → create PR → merge → tag" release pattern. The workflow eliminates local `gh` CLI dependencies and supports both github.com and GitHub Enterprise instances.

### Stakeholders
- Developers who use squash-merge-release workflow
- CI/CD pipelines requiring automated releases
- Teams using GitHub Enterprise

### Constraints
- Dagger functions are non-interactive; all parameters must be explicit
- Source directory must be explicitly provided (`--source=.`)
- User must extract GitHub token from `~/.config/gh/hosts.yaml` before calling
- Only HTTPS remotes supported for git push authentication

## Goals / Non-Goals

### Goals
- Eliminate local `gh` CLI installation requirements
- Support both github.com and GitHub Enterprise via token-based auth
- Provide clear, actionable error messages
- Single command to execute entire workflow
- Automatic VERSION file detection for tagging

### Non-Goals
- SSH key authentication for git push
- Interactive prompts or user input during execution
- GitHub Actions integration
- Multi-remote support
- Automatic local state synchronization after merge

## Decisions

### Decision 1: Use `ghcr.io/cli/cli:latest` container image
- **Why**: Official GitHub CLI image, always up-to-date, includes all `gh` dependencies
- **Alternatives considered**:
  - Alpine with manual `gh` installation: More control but maintenance burden
  - Custom image: Unnecessary complexity for this use case

### Decision 2: Token passed as `dagger.Secret`
- **Why**: Dagger's native secret handling prevents accidental exposure in logs/cache
- **Usage**: `--github-token=env:GITHUB_TOKEN` pattern works naturally
- **Alternatives considered**:
  - Direct environment variable: Less secure, visible in logs
  - File-based token: More complex, requires mounting

### Decision 3: Auto-detect GH_HOST from git remote
- **Why**: Reduces user friction; don't require explicit host for common github.com case
- **Implementation**: Parse `origin` remote URL, extract host from SSH (`git@host:`) or HTTPS (`https://host/`) format
- **Override**: Allow explicit `--gh-host` parameter when auto-detection fails

### Decision 4: Git config for commits
- **Why**: Container has no git identity; commits require user.email and user.name
- **Values**: `user.email=dagger@localhost`, `user.name=Dagger Workflow`
- **Rationale**: Generic identity clearly indicates automated commits

### Decision 5: VERSION file detection pattern
- **Why**: Support common project layouts
- **Locations checked** (in order):
  1. `VERSION` (root)
  2. `version/VERSION`
- **Behavior**: If no VERSION file, skip tagging (no error)

### Decision 6: HTTPS token injection for git push
- **Why**: SSH keys require complex setup; HTTPS with token is simpler in containers
- **Implementation**: Use git credential helper or URL rewriting with token embedded
- **Security**: Token in memory only, not persisted to credential cache

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Dagger Module                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  GHRelease Class                     │   │
│  │                                                      │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │   │
│  │  │   validate   │  │  get_version │  │ _gh_exec  │  │   │
│  │  │   (public)   │  │   (public)   │  │ (private) │  │   │
│  │  └──────────────┘  └──────────────┘  └───────────┘  │   │
│  │                                                      │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │           squash_merge_release                │   │   │
│  │  │               (main function)                 │   │   │
│  │  │                                               │   │   │
│  │  │  1. Validate branch/commits                   │   │   │
│  │  │  2. Squash commits                            │   │   │
│  │  │  3. Force push to feature branch              │   │   │
│  │  │  4. Create PR                                 │   │   │
│  │  │  5. Merge PR with squash                      │   │   │
│  │  │  6. Create version tag (if VERSION exists)   │   │   │
│  │  │  7. Return result summary                     │   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
│                            ▼                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              ghcr.io/cli/cli:latest                 │   │
│  │  - gh pr create / merge                             │   │
│  │  - gh release create (for tags)                     │   │
│  │  - git operations via container                     │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Risks / Trade-offs

| Risk | Impact | Mitigation |
|------|--------|------------|
| Token exposure in logs | High | Use `dagger.Secret`, never echo token |
| Force push to wrong branch | High | Double-check branch name before push |
| GH_HOST auto-detection fails | Medium | Provide clear error with override option |
| VERSION file in unexpected location | Low | Document supported locations |
| Container image breaks compatibility | Low | Pin to specific version if needed |

## Migration Plan

Not applicable - this is a new module with no existing implementation to migrate.

## Open Questions

1. **Should we support base branch configuration?**
   - Currently assumes `main` with `master` fallback
   - Could add `--base-branch` parameter if needed

2. **Should we support dry-run mode?**
   - Would show what operations would be performed without executing
   - Useful for validation before actual release

3. **Should we support multiple VERSION file formats?**
   - Currently plain text (e.g., `1.0.0`)
   - Could support `version.json`, `package.json`, etc.