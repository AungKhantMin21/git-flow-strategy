# Git Flow Strategy

> A hybrid **Git Flow + Permanent UAT Branch** model with two deployment environments: **UAT** and **Production**.

---

## 1. Branch Structure

| Branch | Purpose | Deploys to | Protected |
|---|---|---|---|
| `main` | Production-ready code. Always stable. | **Production** | Yes |
| `uat` | Permanent UAT environment branch. Reflects exactly what stakeholders are testing. | **UAT** | Yes |
| `develop` | Integration branch. All features land here first. | — | Yes |
| `feature/*` | Individual features or non-urgent fixes. | Local only | No |
| `hotfix/*` | Emergency production fixes only. | UAT (quick verify) | No |

### Branch naming

| Type | Pattern | Example |
|---|---|---|
| Feature | `feature/<short-description>` | `feature/invoice-export` |
| Feature with ticket | `feature/<TICKET-ID>-<description>` | `feature/PROJ-42-user-search` |
| Hotfix | `hotfix/<semver>-<description>` | `hotfix/1.2.1-payment-crash` |

> Lowercase and hyphens only. No spaces, underscores, or special characters.

---

## 2. Environment Map

```
feature/*  →  local only
               │
               │  PR approved → merge
               ▼
           develop   (integration — all features accumulate here)
               │
               │  PR approved → merge → auto deploy
               ▼
            uat       (permanent — tagged uat-X.Y.Z on every merge)
               │
               │  PR approved after stakeholder sign-off → merge → auto deploy
               ▼
            main      (production — tagged vX.Y.Z on every merge)
```

---

## 3. Flow Diagrams

### 3.1 Feature Flow

Branch from `develop`, work, open a PR back to `develop`.

```
develop ──┬──────────────────────────────┬──▶ develop
          │                              │
          └─▶ feature/your-work ─────────┘
               (commit, commit, commit)
               PR reviewed + approved
```

### 3.2 UAT Flow

When `develop` is ready for stakeholder testing, open a PR from `develop` to `uat`. On merge, CI/CD deploys to UAT automatically and a tag is created.

```
develop ──────────────────────────────▶ uat (tag: uat-1.2.0)
         PR: "UAT candidate 1.2.0"         │
                                           │  auto deploy
                                           ▼
                                       UAT server
```

If stakeholders find a bug, **fix it on `develop` first** (via a new `feature/*` PR), then open a new PR from `develop` to `uat`. Never commit directly to `uat`.

```
develop ──▶ feature/uat-fix ──▶ develop ──▶ uat (tag: uat-1.2.1)
```

### 3.3 Production Flow

After stakeholder sign-off on UAT, open a PR from `uat` to `main`. On merge, CI/CD deploys to production and a version tag is created.

```
uat (uat-1.2.0) ──────────────────────▶ main (tag: v1.2.0)
                 PR: "Release 1.2.0"         │
                 sign-off confirmed           │  auto deploy
                                             ▼
                                        Production
```

### 3.4 Hotfix Flow

Production emergency. Branch from `uat`, fix it, merge to `uat` for quick test, then merge to both `main` and `develop`.

```
uat ──▶ hotfix/1.2.1-payment-crash ──┬──▶ uat   (quick test)
                                      │
                                      ├──▶ main   (tag: v1.2.1) → deploy
                                      │
                                      └──▶ develop (keep in sync)
```

> Critical: hotfix must merge into **both** `main` and `develop`. Skipping `develop` means the fix is lost in the next release.

---

## 4. Step-by-Step Workflows

### 4.1 Feature Development

```bash
# Sync develop first
git checkout develop
git pull origin develop

# Branch
git checkout -b feature/invoice-export

# Work and commit
git commit -m "feat: add invoice PDF generation"
git commit -m "feat: add download button on orders page"
git commit -m "test: add unit tests for PDF service"

# Push and open PR → target: develop
git push origin feature/invoice-export

# Open PR request in GitHub/GitLab to merge into develop
# After code review and approval — merge PR into develop

# After merge, sync develop
git checkout develop
git pull origin develop

# Clean up feature branch
git branch -d feature/invoice-export
git push origin --delete feature/invoice-export
```

---

### 4.2 UAT Deploy

```bash
# Ensure develop is stable and has everything needed for this UAT cycle
git checkout develop
git pull origin develop

# Open a PR → target: uat
# Title: "UAT candidate 1.2.0"
# After PR approval — merge into uat

# Tag and push to uat
git tag -a uat-1.2.0 -m "UAT build 1.2.0"
git push origin uat --tags
# → CI/CD deploys to UAT automatically

# If a bug is found during UAT testing:
# 1. Fix it on develop via a normal feature/* PR (section 4.1)
# 2. Then open a new PR and merge from develop → uat
# 3. Tag and push to uat
git tag -a uat-1.2.1 -m "UAT build 1.2.1"
git push origin uat --tags
```

> Rule: `uat` is a receiving branch only. Nobody commits directly to it.

---

### 4.3 Production Release

```bash
# UAT sign-off received
# Open a PR → target: main
# Title: "Release 1.2.0"

# After PR approval — merge into main

# Tag and push to main
git tag -a v1.2.0 -m "Release version 1.2.0"
git push origin main --tags
# → CI/CD deploys to Production
```

---

### 4.4 Hotfix

```bash
# Branch from main (NOT develop)
git checkout main
git pull origin main
git checkout -b hotfix/1.2.1-payment-crash

# Fix
git commit -m "fix: handle null user session in payment gateway"
git push origin hotfix/1.2.1-payment-crash

# Open a PR and merge from hotfix/1.2.1-payment-crash → uat
# Tag and push to uat
git tag -a uat-1.2.1 -m "UAT build 1.2.1"
git push origin uat --tags

# After smoke test passes:

# Open a PR and merge to main
# Tag and push to main
git tag -a v1.2.1 -m "Hotfix 1.2.1"
git push origin main --tags

# ***CRITICAL***: also merge to develop

# Clean up
git branch -d hotfix/1.2.1-payment-crash
git push origin --delete hotfix/1.2.1-payment-crash
```

---

## 5. Rollback Procedures

All protected branches (`main`, `uat`, `develop`) cannot be pushed to directly.
Rollback uses a `hotfix/*` branch as the vehicle — revert in the hotfix branch,
then PR it into the target branch.

### How it works
```
hotfix/rollback-* branch
    │
    ├── revert commits made here
    │
    ├── PR → uat      → tag uat-X.Y.Z-rollback
    ├── PR → main     → tag vX.Y.Z-rollback  
    └── PR → develop
```

> Rule: always start from `uat`, let it flow to `main` via PR,
> then handle `develop` separately. Never start from `main`.

---

### Workflow

Use when a bad release is live on production. Rolls back both `uat` and `main`.
```bash
# Step 1 — create rollback branch from uat
git checkout uat && git pull origin uat
git checkout -b hotfix/rollback-vX.Y.Z

# Step 2 — 
# Option A) Revert using tag range
git revert goodVersion..badVersion
# e.g. git revert v1.0..v1.1

# Option B) Revert using merge commit
git revert -m 1 <merge-commit-hash>

# Option C) Revert using commit sha
git revert <commit-sha>

# Step 3 — push the hotfix branch
git push origin hotfix/rollback-vX.Y.Z
```
```bash
# Step 4 — PR: hotfix/rollback-vX.Y.Z → uat
# After merge, tag uat:
git checkout uat && git pull origin uat
git tag -a uat-X.Y.Z-rollback -m "Rollback to X.Y.Z state"
git push origin uat --tags
# → CI/CD redeploys UAT
```
```bash
# Step 5 — PR: uat → main
# After merge, tag main:
git checkout main && git pull origin main
git tag -a vX.Y.Z-rollback -m "Rollback to X.Y.Z state"
git push origin main --tags
# → CI/CD redeploys Production
```
```bash
# Step 6 — sync develop separately
# PR from hotfix/rollback-vX.Y.Z to develop
# After merge:
git checkout develop && git pull origin develop
```
```bash
# Step 7 — clean up hotfix branches
git branch -d hotfix/rollback-vX.Y.Z
git push origin --delete hotfix/rollback-vX.Y.Z
```
---

## 6. Tag Convention

All tags must be following semantic versioning pattern.

`Major.minor.patch`



| Tag | Branch |
|---|---|
| `uat-1.2.0` | `uat` |
| `uat-1.2.1` | `uat` |
| `v1.2.0` | `main` |
| `v1.2.1` | `main` |

Tags are the audit trail. Given any tag, you can always check out the exact code that was running at that point.

```bash
git checkout uat-1.2.0    # what stakeholders approved
git checkout v1.2.0       # what went to production
```

---

## 7. Branch Protection Rules

| Branch | Rules |
|---|---|
| `main` | Require PR · Require 1 approval · Block direct push · Block force push |
| `uat` | Require PR · Block direct push · Block force push |
| `develop` | Require PR · Require 1 approval · Block direct push |

---

## 8. Golden Rules

1. **Never commit directly** to `main`, `uat`, or `develop` — always via PR.
2. **`uat` only receives from `develop` or `hotfix/*`** — never from feature branches directly.
3. **Hotfix merges into three branches**: `main`, `uat`, and `develop`.
4. **Fix UAT bugs on `develop` first**, then re-PR to `uat`.
5. **Tag every merge** to `uat` and `main` — no exceptions.
6. **Delete feature and hotfix branches** after merging for cleaner repository.
