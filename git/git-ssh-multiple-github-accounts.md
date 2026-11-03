# Git over SSH with several GitHub accounts: "Permission denied to <other-user>", host aliases, and why IntelliJ picks a different key than the terminal

One machine, two or more GitHub accounts (personal, employer, a client's organisation), each with its own SSH key. Pushes work from the terminal but fail from an IDE, or work in one repo and fail in another, with GitHub naming the **wrong account** in the error. The cause is always the same: nothing tells SSH which key belongs to which repository, so each client guesses, and they guess differently. Companion docs: [git-cli-explained.md](git-cli-explained.md) (remotes and `git remote set-url`), [ssh-tunneling.md](../infrastructure-ops/ssh-tunneling.md) (the same `~/.ssh/config` mechanics for port forwarding), [cloud-manager-git-repositories.md](../cloud-service/cloud-manager-git-repositories.md) (how a host alias makes a plain github.com repo look like a different host).

## The symptom

```
ERROR: Permission to alice/aem-knowledgebase.git denied to alice-work.
fatal: Could not read from remote repository.
```

Read the two names. The repository belongs to `alice`, but GitHub authenticated the connection as `alice-work`. Authentication **succeeded**; it just succeeded as the wrong person. "Make sure you have the correct access rights" is misleading: the key that was offered does not belong to an account with access, but another key on the same machine does.

Confirm which identity each client uses:

```
ssh -T git@github.com              # "Hi alice! You've successfully authenticated..."
ssh -T git@github.personal         # against an alias, see below
```

## Why the two clients disagree

GitHub identifies you purely by **which SSH key** signs the connection; the `git@` user is the same for everyone. When a remote is `git@github.com:alice/repo.git` and `~/.ssh/config` has no `Host github.com` entry, SSH offers keys in an order you did not choose: keys loaded in `ssh-agent` first, in load order, then the default `~/.ssh/id_*` files. GitHub accepts the **first key that matches any account**, and from then on you *are* that account for this connection.

Two clients on one machine do not share that order:

- The **terminal** uses the native `ssh` and the agent, in whatever order keys were added (often the key you used most recently).
- **IntelliJ, PyCharm, WebStorm** and the other JetBrains IDEs default to a **built-in SSH client** (Settings → Version Control → Git → SSH executable: *Built-in*). It reads `~/.ssh/config` only partially, has its own key discovery order, and asks the agent in its own way. It will frequently pick the work key where the terminal picked the personal one.
- The macOS keychain, `AddKeysToAgent yes`, and a key added for a different repo an hour ago all change the agent's order silently. "It worked yesterday" is normal.

So the terminal succeeding proves nothing about the IDE, and vice versa.

## The fix: make the key explicit with a host alias

Give every account its own alias in `~/.ssh/config`, with `IdentitiesOnly yes` so SSH offers **only** that key and never falls back to the agent's list:

```
# ~/.ssh/config
Host github.personal
  HostName github.com
  User git
  IdentitiesOnly yes
  IdentityFile ~/.ssh/id_ed25519_personal
  AddKeysToAgent yes

Host github.work
  HostName github.com
  User git
  IdentitiesOnly yes
  IdentityFile ~/.ssh/id_ed25519_work
  AddKeysToAgent yes
```

Then point each repository's remote at the alias instead of the bare host:

```
git remote -v                                             # git@github.com:alice/repo.git
git remote set-url origin git@github.personal:alice/repo.git
ssh -T git@github.personal                                # Hi alice!
git push
```

The alias is only an SSH name; GitHub still sees `github.com`. Every client, terminal or IDE, now resolves `github.personal` through the same config block and the same single key, so the identity cannot drift. Clone new repositories with the alias from the start:

```
git clone git@github.work:example-org/repo.git
```

`IdentitiesOnly yes` is the important line. Without it SSH still offers the agent's keys *before* the `IdentityFile`, and the wrong account wins again.

## Alternatives, and why the alias is preferred

| Approach | Works | Drawback |
|---|---|---|
| **Host alias per account** (above) | Everywhere, every client | Remote URLs look unusual (`github.personal:`); tooling that parses `github.com` out of the URL may not recognise it |
| `Host github.com` block pinning one key | Terminal and IDE | Only one account can use plain `github.com`; the others still need aliases |
| Per-directory `includeIf` in `~/.gitconfig` with `core.sshCommand = ssh -i ~/.ssh/id_work` | Terminal | JetBrains built-in SSH ignores `core.sshCommand`; set **SSH executable: Native** for it to apply |
| Switch IDE to **Native** SSH executable | Makes the IDE behave like the terminal | Only moves the guess to the native client; still unordered without an alias or pinned host block |
| HTTPS remotes with a credential helper (`gh auth`) | Everywhere | Credential helper is keyed per host, so two accounts on `github.com` need `credential.useHttpPath` and per-repo tokens; usually worse than SSH aliases |

## Diagnosing in one minute

1. Read the error: **which user was denied**. That is the key that won.
2. `git remote -v`: is the host bare `github.com`? Then nothing pins the key.
3. `ssh -T git@github.com` from the terminal and compare with the IDE's user. Different answers confirm the two clients are guessing.
4. `ssh-add -l` to see the agent's key order; `ssh -vT git@github.com 2>&1 | grep -i 'offering\|accepted'` shows which key was offered and accepted.
5. Add or reuse an alias with `IdentitiesOnly yes`, `git remote set-url origin git@<alias>:owner/repo.git`, push.

## Gotchas

- **Both accounts cannot hold the same key.** GitHub rejects adding a public key that another account already uses; one key per account is mandatory, which is why multi-account always means multiple keys.
- **Deploy keys are repo-scoped**, so a deploy key is a third kind of identity that also needs its own alias.
- **Submodules and `go get`-style tooling** build URLs with `github.com` literally; an alias in the parent remote does not propagate. Use a `url.<alias>.insteadOf` rewrite in `~/.gitconfig` for those: `git config --global url."git@github.work:example-org/".insteadOf "git@github.com:example-org/"`.
- **The JetBrains IDE caches the result.** After changing the remote or the SSH config, if the IDE still names the old user, invalidate via a fresh Fetch or restart; the built-in client keeps a session open.
- **Keychain prompts on macOS** come from `UseKeychain yes` and `AddKeysToAgent yes`; they are convenient but they are what reorders the agent over time.
- **The same pattern applies to Bitbucket, GitLab and Azure DevOps**: all identify you by key, all accept the first matching one, all are fixed by a host alias.

## What this means for AEM

- **Cloud Manager Git** uses HTTPS with per-user credentials, not SSH keys, so it never has this problem; but a project that mirrors Cloud Manager to GitHub (see [cloud-manager-git-repositories.md](../cloud-service/cloud-manager-git-repositories.md)) typically lives in a **client's** GitHub organisation under a client-issued account, which is exactly the second-identity case. Give it an alias on day one.
- **Agency laptops** carry a personal account, the employer's, and one per client engagement; a push that lands under the wrong identity can leak a commit author or fail a client's SSO-protected organisation with the same "denied to" error. The alias also documents which identity a checkout belongs to: `git remote -v` answers it.
- **CI runners and build boxes** for AMS deployments usually hold one deploy key or one machine user; the ordering problem appears when someone adds a second key "temporarily" to the same runner.

## Cheat sheet

- `Permission to X denied to Y`: authenticated as the **wrong account** `Y`, not unauthenticated. Which key won is the whole question.
- Bare `github.com` remote + no `Host github.com` block = SSH offers agent keys in arbitrary order; terminal and JetBrains built-in SSH pick differently.
- Fix: per-account `Host github.<name>` alias with `HostName github.com`, `IdentitiesOnly yes`, `IdentityFile`, then `git remote set-url origin git@github.<name>:owner/repo.git`.
- `IdentitiesOnly yes` is mandatory; without it the agent's keys are still tried first.
- Verify with `ssh -T git@github.<name>`; the greeting names the account.
- JetBrains: built-in SSH ignores `core.sshCommand`; set SSH executable to Native if relying on it, but an alias works with either.
- Submodules and tooling that hard-code `github.com`: `url.<alias>.insteadOf` rewrite.
