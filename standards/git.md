# Git Workflow (2025)

> **Last updated**: January 2025
> **Scope**: Branches, commits, PRs, tags

---

## Philosophy

Git history should tell a story. Each commit is a logical, atomic change that can be understood in isolation.

---

## Branch Naming

| Type | Pattern | Example |
|------|---------|---------|
| **Feature** | `feat/<ticket>-<description>` | `feat/PROJ-123-user-auth` |
| **Bugfix** | `fix/<ticket>-<description>` | `fix/PROJ-456-login-error` |
| **Hotfix** | `hotfix/<description>` | `hotfix/critical-security-patch` |
| **Release** | `release/<version>` | `release/1.2.0` |
| **Chore** | `chore/<description>` | `chore/update-dependencies` |

### Rules

- Always lowercase
- Use hyphens, not underscores
- Keep it short but descriptive
- Include ticket number when applicable

---

## Commit Messages

### Conventional Commits

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

### Types

| Type | When to use |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `style` | Formatting, no code change |
| `refactor` | Code change, no new feature or fix |
| `perf` | Performance improvement |
| `test` | Adding/fixing tests |
| `chore` | Build, tooling, deps |
| `ci` | CI/CD changes |
| `revert` | Revert previous commit |

### Examples

```bash
feat(auth): add OAuth2 login with Google

fix(api): handle null response from payment provider

docs(readme): update installation instructions

refactor(user): extract validation logic to separate module

chore(deps): update Next.js to 15.1.0

BREAKING CHANGE: remove deprecated /api/v1 endpoints
```

### Rules

- Imperative mood ("add" not "added")
- No period at the end
- Max 72 characters for subject line
- Body for "what" and "why", not "how"

---

## Branching Strategy

### GitHub Flow (Recommended for most)

```
main ──────────────────────────────────────────►
       \                    /
        feat/user-auth ────►
                    (PR + merge)
```

- `main` is always deployable
- Create feature branches from `main`
- Open PR when ready
- Merge via squash or merge commit
- Delete branch after merge

### Trunk-Based (For advanced teams)

- Very short-lived branches (< 1 day)
- Feature flags for incomplete work
- Continuous deployment from main

---

## Pull Requests

### PR Title

Follow commit convention:
```
feat(auth): add OAuth2 login with Google
```

### PR Description Template

```markdown
## What

Brief description of changes.

## Why

Context and motivation.

## How

Implementation details if complex.

## Testing

- [ ] Unit tests added
- [ ] E2E tests added
- [ ] Manual testing done

## Screenshots

If UI changes.
```

### Rules

- Keep PRs small (< 400 lines ideal)
- One logical change per PR
- Self-review before requesting review
- Resolve all comments before merge

---

## Tags & Releases

### Semantic Versioning

```
v<major>.<minor>.<patch>

v1.0.0  - Initial release
v1.1.0  - New feature (backward compatible)
v1.1.1  - Bug fix
v2.0.0  - Breaking change
```

### Creating a Release

```bash
git tag -a v1.2.0 -m "Release 1.2.0"
git push origin v1.2.0
```

---

## Quick Reference

| Action | Command |
|--------|---------|
| Create branch | `git checkout -b feat/description` |
| Stage all | `git add .` |
| Commit | `git commit -m "feat: description"` |
| Push | `git push -u origin feat/description` |
| Update from main | `git pull origin main --rebase` |
| Squash last N | `git rebase -i HEAD~N` |
| Amend last | `git commit --amend` |
| Undo last commit | `git reset --soft HEAD~1` |
