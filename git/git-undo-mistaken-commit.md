# Undoing a mistaken commit without touching the files

A commit was made too early, with the wrong message, or by accident, and the goal is to make it disappear while every file stays exactly as it is on disk and in the staging area. Git has one command for the normal case and a different one for the one case where it fails: the very first commit of a repository.

## First check: has it been pushed?

```bash
git log --oneline -3          # what is on the branch
git branch -r                 # remote-tracking branches (empty = nothing fetched/pushed yet)
git ls-remote --heads origin  # what the remote actually has
```

Everything below rewrites local history. That is harmless while the commit exists only on your machine. Once it has been pushed and someone may have pulled it, do not rewrite; add a new commit that undoes it instead:

```bash
git revert <commit>
```

## Normal case: the commit has a parent

```bash
git reset --soft HEAD~1
```

`reset` moves the branch pointer; the mode decides what happens to the index and working tree:

| Mode | Branch pointer | Index (staged) | Working tree |
|---|---|---|---|
| `--soft` | moved back | **kept as it was in the commit** | kept |
| `--mixed` (default) | moved back | reset to the new HEAD, so files show as unstaged | kept |
| `--hard` | moved back | reset | **reset, work discarded** |

`--soft` is the "undo the commit, keep everything staged" option: `git status` afterwards looks exactly like it did the moment before `git commit`.

## Root-commit case: `HEAD~1` does not exist

The first commit of a repository has no parent, so `HEAD~1` is not a valid revision and `git reset --soft HEAD~1` fails with:

```
fatal: ambiguous argument 'HEAD~1': unknown revision or path not in the working tree.
```

The equivalent for a root commit is to delete the branch pointer itself:

```bash
git update-ref -d HEAD
```

This removes the ref the current branch points to, so the branch goes back to "no commits yet". The index and working tree are not touched: `git status` shows every file as staged (`A`), exactly as before the commit. `git log` now reports that the branch has no commits, which is the expected result, not an error.

Recovery, if needed: the deleted commit still exists as an object until garbage collection. `git reflog` will not list it (the ref is gone), but the hash printed by the original `git commit` output, or `git fsck --lost-found`, lets you `git reset --soft <hash>` back onto it.

## Related

- Mental model of working tree, index and repository, and the full `reset` / `restore` / `revert` family: [[git-cli-explained]].
- Fixing only the message or adding a forgotten file to the last commit is `git commit --amend`, not a reset.

## What this means for AEM

Nothing AEM-specific; this is plain Git. It comes up in AEM work when a fresh dispatcher-config lab, a Cloud Manager repo clone, or an archetype-generated project gets its first commit before the files are ready. Cloud Manager pipelines build from branches you push, so an unpushed local undo is invisible to Cloud Manager; a pushed commit on a pipeline branch must be reverted, never rewritten, because Cloud Manager may already have built it ([[cloud-manager-git-repositories]]).
