# Git CLI — the mental model and the commands you actually use

Git is a distributed version control system; the `git` command is its interface. **Every clone is a full repository with complete history**, so nearly every operation is local and fast, and a remote is just another copy you sync with. This page is the orientation layer; the AEM-specific Git mechanics (Cloud Manager repos, credentials, multi-repo setups) live in the linked docs.

## Mental model: three areas and two pointers

| Area | Holds | Moved by |
|---|---|---|
| **Working tree** | the files on disk you edit | editor, `git restore`, `git checkout` |
| **Index** (staging area) | what the *next commit* will contain | `git add`, `git restore --staged`, `git reset` |
| **Repository** (`.git/`) | commits: immutable snapshots pointing to their parents | `git commit`, `git merge`, `git rebase` |

- A **branch** is a movable pointer to a commit; creating one costs nothing.
- **HEAD** points to the branch you are on (or directly to a commit when "detached").
- A **remote** (`origin` by default) is a named URL plus remote-tracking branches like `origin/main`, which only move on `fetch`/`push`.

## Daily commands

- **Start**: `git init`, `git clone <url>`.
- **Look**: `git status`, `git diff` (tree vs index), `git diff --staged` (index vs HEAD), `git log --oneline --graph --all`, `git show <commit>`, `git blame <file>`.
- **Record**: `git add <path>`, `git add -p` (hunk by hunk), `git commit -m "…"`, `git commit --amend` (fix the last commit — local only).
- **Branches**: `git switch <branch>`, `git switch -c <new>`, `git branch -d <merged>`. `git checkout` is the older command that does both branch switching and file restore.
- **Sync**: `git fetch` (download, touch nothing), `git pull` (= fetch + merge, or fetch + rebase with `--rebase`), `git push`, `git push -u origin <branch>` the first time.
- **Combine**: `git merge <branch>` (merge commit, history preserved), `git rebase <base>` (replay your commits on a new base, linear history), `git cherry-pick <commit>` (copy one commit).
- **Park**: `git stash`, `git stash pop`, `git stash list`.
- **Undo**: `git restore <path>` (discard tree changes), `git restore --staged <path>` (unstage), `git revert <commit>` (undo via a *new* commit — the safe choice on shared branches), `git reset --soft|--mixed|--hard <commit>` (move the branch pointer; `--soft` keeps index + tree, `--mixed` keeps tree, `--hard` keeps nothing).
- **Mark and find**: `git tag -a v1.0 -m "…"`, `git bisect start|good|bad` (binary-search history for the breaking commit), `git reflog` (where HEAD has been — the recovery tool).

## Configuration

- Identity: `git config --global user.name "…"` and `user.email "…"` — stamped on every commit.
- `.gitignore` lists untracked paths to skip; `.gitattributes` sets per-path behaviour (line endings, LFS, diff drivers).
- `git remote -v`, `git remote add <name> <url>`; a second remote is how a mirror to Cloud Manager's Adobe-hosted repo works.
- `git config --global pull.rebase true` if the team wants linear history; otherwise `pull` makes merge commits.

## Gotchas

- **Never rewrite history others have pulled.** Rebasing, amending or force-pushing a shared branch forces everyone else to recover. `git push --force-with-lease` at least refuses to clobber commits you have not seen.
- **`git reset --hard` and `git clean -fd` destroy uncommitted work with no undo.** `git reflog` recovers lost *commits*; nothing recovers unstaged changes.
- `git pull` onto a dirty tree fails or produces surprise merges. Commit or stash first.
- Line endings: Windows CRLF vs Unix LF turns every line into a diff. Fix with `core.autocrlf` or `* text=auto` in `.gitattributes`.
- Large binaries bloat history forever — every clone downloads every version. Use **Git LFS** for them.
- A detached HEAD (checked out a tag or commit) loses commits when you switch away unless you create a branch first.
- Submodules are a pointer to a commit in another repo, not a copy; `git clone --recurse-submodules` or `git submodule update --init` or you get empty folders.

## AEM consequences

- Cloud Manager pipelines build from a **branch** you select; tags and releases are generated Adobe-side and must be synced back if you mirror — [[cloud-manager-git-repositories]].
- The Adobe-hosted repo uses **Git-specific credentials from Access Repo Info**, not your Adobe ID; a regenerated password invalidates the old one everywhere, which is the usual "sudden auth failure" — [[cloud-manager-git-repositories]].
- Bring-your-own GitHub is public github.com only; GitHub Enterprise and GitLab go through an *external* repository — [[cloud-manager-git-repositories]].
- Several team repos feeding one Cloud Manager repo is either CI-side aggregation (with a `git add --all` deletion trap) or Git submodules — [[cloud-manager-multi-repo-team-setup]].
- Never commit `.content.xml` files with environment-specific values or secrets; Cloud Manager variables exist for that — [[cloud-manager-variables]].
