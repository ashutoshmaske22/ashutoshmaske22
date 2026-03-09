# 🌿 Git & GitOps

## Git Core Concepts

```
Working Dir → Staging Area (index) → Local Repo → Remote Repo
              git add                git commit   git push
```

---

## 📋 Git Cheatsheet

### Everyday Workflow
```bash
git status
git diff                         # unstaged changes
git diff --staged                # staged changes
git add -p                       # interactive staging (chunk by chunk)
git commit -m "feat: add health endpoint"
git commit --amend --no-edit     # add to last commit (don't push)
```

### Branching
```bash
git checkout -b feature/add-auth    # create + switch
git switch -c feature/add-auth      # modern syntax
git branch -d feature/add-auth      # delete local
git push origin --delete feature/add-auth  # delete remote
git branch -vv                      # show tracking branches
```

### Merging vs Rebasing
```bash
# Merge (preserves history, creates merge commit)
git merge feature/add-auth

# Rebase (linear history, rewrites commits)
git rebase main
git rebase -i HEAD~3    # interactive rebase — squash, reword, reorder

# Golden rule: NEVER rebase shared/public branches
```

### Stash
```bash
git stash                      # stash uncommitted changes
git stash push -m "WIP: auth"  # named stash
git stash list
git stash pop                  # apply + drop latest
git stash apply stash@{2}      # apply specific, keep stash
```

### Undoing Things
```bash
git restore file.py              # discard working dir changes
git restore --staged file.py     # unstage
git reset HEAD~1                 # undo last commit, keep changes staged
git reset --hard HEAD~1          # undo + discard changes (DANGEROUS)
git revert abc1234               # create new commit reversing abc1234 (safe for shared branches)
git reflog                       # history of HEAD — recover from disasters
```

### Log & Search
```bash
git log --oneline --graph --all   # pretty branch graph
git log --author="Alice" --since="1 week ago"
git log -S "function_name"        # commits that added/removed this string
git blame file.py                 # who changed each line
git bisect start / bad / good     # binary search for bug-introducing commit
```

---

## Branching Strategies

### GitHub Flow (simple, for CD)
```
main (always deployable)
  └── feature/xxx → PR → merge to main → auto-deploy
```

### Git Flow (complex releases)
```
main         → production releases
develop      → integration branch
feature/*    → branch from develop
release/*    → branch from develop for release prep
hotfix/*     → branch from main for urgent fixes
```

### Trunk-Based Development (Google/Meta style)
```
main         → one main branch, very short-lived feature branches (<1 day)
               feature flags control what's visible in production
```

---

## GitOps

### What is GitOps?
> Git is the **single source of truth** for infrastructure and application state.  
> A controller (ArgoCD/Flux) continuously syncs the cluster to match what's in Git.

```
Developer pushes → Git repo → ArgoCD detects drift → applies to K8s → cluster matches Git
```

### ArgoCD Basics
```yaml
# Application manifest
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/gitops-repo
    targetRevision: main
    path: apps/myapp/overlays/prod   # Kustomize
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # delete resources removed from Git
      selfHeal: true     # revert manual kubectl changes
    syncOptions:
      - CreateNamespace=true
```

```bash
# ArgoCD CLI
argocd app list
argocd app sync myapp
argocd app rollback myapp 2    # rollback to revision 2
argocd app history myapp
```

---

## 🛠️ Hands-On Project: GitOps with ArgoCD

**What you'll build:** A multi-environment (dev/staging/prod) GitOps setup with Kustomize overlays and ArgoCD.

```
gitops-repo/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml     # patch: 1 replica, dev image
    ├── staging/
    │   └── kustomization.yaml     # patch: 2 replicas
    └── prod/
        └── kustomization.yaml     # patch: 3 replicas, resource limits
```

---

## 🎯 Key Interview Questions

1. **What's the difference between `git merge` and `git rebase`?**  
   Merge preserves full branch history with a merge commit. Rebase replays commits on top of another branch for a linear history.

2. **How do you recover a accidentally deleted branch?**  
   `git reflog` to find the last commit SHA, then `git checkout -b branch-name <sha>`.

3. **What is GitOps and how does it differ from traditional CI/CD?**  
   GitOps uses Git as the source of truth; a controller (ArgoCD) reconciles cluster state to Git continuously. Traditional CD pushes changes; GitOps pulls them.

4. **What is the difference between `git reset` and `git revert`?**  
   `reset` rewrites history (unsafe for shared branches). `revert` creates a new commit undoing changes — safe for shared branches.

5. **What is a merge conflict and how do you resolve it?**  
   Occurs when two branches change the same lines differently. Resolve by editing conflict markers, `git add`, then `git commit`.
