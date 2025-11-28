# Implementation Tasks

## 1. Module Setup

- [ ] 1.1 Initialize Python SDK: `dagger init --sdk=python` (if not already done)
- [ ] 1.2 Update `dagger.json` with SDK configuration
- [ ] 1.3 Create `pyproject.toml` with `uv_build` backend and `name = "main"`
- [ ] 1.4 Add dependencies: `dagger-io`, `requests` (if needed for API calls)
- [ ] 1.5 Create `.gitignore` with `/sdk/`, `__pycache__/`, `.venv/` entries

## 2. Core Module Structure

- [ ] 2.1 Create `src/main/__init__.py` with `DaggerGhRelease` class
- [ ] 2.2 Add `@object_type` decorator to main class
- [ ] 2.3 Verify class name matches dagger.json module name after CamelCase conversion

## 3. Helper Functions (Private)

- [ ] 3.1 Implement `_detect_gh_host(source: Directory)` - parse remote URL for host
- [ ] 3.2 Implement `_get_base_container(source: Directory, github_token: Secret, gh_host: str)` - build container with gh CLI, git, and credentials
- [ ] 3.3 Implement `_read_version_file(source: Directory)` - check VERSION locations
- [ ] 3.4 Implement `_exec_gh(container: Container, args: list)` - execute gh commands with GH_HOST

## 4. Validation Function

- [ ] 4.1 Create `validate(source, github_token, gh_host=None)` public function
- [ ] 4.2 Implement working directory clean check (`git status --porcelain`)
- [ ] 4.3 Implement branch check (not main/master)
- [ ] 4.4 Implement commits ahead check (`git rev-list --count`)
- [ ] 4.5 Implement gh auth status check
- [ ] 4.6 Implement merge conflict detection (`git merge-tree` or `git diff --check`)
- [ ] 4.7 Return structured validation result

## 5. Version Detection Function

- [ ] 5.1 Create `get_version(source)` public function
- [ ] 5.2 Check `VERSION` at root
- [ ] 5.3 Check `version/VERSION` as fallback
- [ ] 5.4 Return version string or empty string

## 6. Main Workflow Function

- [ ] 6.1 Create `squash_merge_release(source, github_token, squash_message, gh_host=None, force_large_squash=False)` public function
- [ ] 6.2 Call validate function first (fail early)
- [ ] 6.3 Implement large commit count guardrail (>100 commits requires `force_large_squash=True`)
- [ ] 6.4 Implement soft reset to main (`git reset --soft main`)
- [ ] 6.5 Implement commit with squash message (`git commit -m`)
- [ ] 6.6 Implement force push (`git push --force-with-lease`)
- [ ] 6.7 Implement PR creation (`gh pr create`)
- [ ] 6.8 Handle existing PR case (`gh pr list --head`)
- [ ] 6.9 Implement PR merge (`gh pr merge --squash --delete-branch`)
- [ ] 6.10 Implement version detection and tag creation
- [ ] 6.11 Build and return success result string (include caution if large squash was forced)

## 7. Git Credential Setup

- [ ] 7.1 Configure git user.email and user.name in container
- [ ] 7.2 Configure git credential helper for HTTPS token auth
- [ ] 7.3 Test push authentication works with token

## 8. Error Handling

- [ ] 8.1 Add clear error messages for dirty working directory
- [ ] 8.2 Add clear error messages for wrong branch
- [ ] 8.3 Add clear error messages for no commits ahead
- [ ] 8.4 Add clear error messages for auth failures
- [ ] 8.5 Add clear error messages for tag exists scenario
- [ ] 8.6 Add clear error messages for merge conflicts (with `git rebase main` instructions)
- [ ] 8.7 Add clear error messages for large commit count (>100) without force flag
- [ ] 8.8 Include best practice guidance about keeping branches closer to main

## 9. Testing

- [ ] 9.1 Run `dagger functions` to verify module loads
- [ ] 9.2 Test `get_version` function with mock source
- [ ] 9.3 Test `validate` function with valid repository state
- [ ] 9.4 Integration test with real test repository (github.com)
- [ ] 9.5 Integration test with GitHub Enterprise (if available)

## 10. Documentation

- [ ] 10.1 Update README.md with usage examples
- [ ] 10.2 Document required token permissions (repo, workflow)
- [ ] 10.3 Add examples for github.com and GitHub Enterprise
- [ ] 10.4 Document local cleanup steps after workflow

## Dependencies / Parallelization

- Tasks 1.x and 2.x are sequential prerequisites
- Tasks 3.x can be done in parallel after 2.x
- Task 4.x depends on 3.1, 3.2, 3.4
- Task 5.x depends on 3.3
- Task 6.x depends on 4.x and 5.x
- Task 7.x can be done in parallel with 4.x, 5.x
- Task 8.x integrated throughout 4.x-6.x
- Task 9.x depends on all implementation tasks
- Task 10.x can start after 6.x is complete

## Acceptance Verification

After implementation, verify all acceptance criteria from OPENSPEC_PROMPT.md:

- [ ] Module builds and passes `dagger functions` check
- [ ] `squash_merge_release` function accepts correct parameters
- [ ] Works with github.com
- [ ] Works with GitHub Enterprise
- [ ] Fails with clear error if: on main branch, no commits ahead, or token invalid
- [ ] Creates and merges PR successfully
- [ ] Creates version tag if VERSION file present
- [ ] Returns structured success message with PR URL and tag URL