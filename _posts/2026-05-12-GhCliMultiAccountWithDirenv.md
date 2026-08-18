---
layout: post
title: Switching git and GitHub CLI identities per directory with direnv
---

I have multiple GitHub identities for different organizations and I keep their clones in parallel trees:

```
~/gh-acc/org1/
~/gh-acc/org2/
...
```

Two pieces need to know about identity when I work in one of those trees: `git` itself (so commits land under the right name and pushes use the right SSH key) and `gh` (so API calls — issues, PRs, releases, Dependabot alerts — authenticate as the right GitHub user). Git has a well-established mechanism for this and ships it out of the box. `gh` doesn't, but direnv plus `GH_TOKEN` gets you the equivalent. The rest of this post covers both halves: git first, then gh.

## Git: per-tree identity with `includeIf gitdir:`

Git supports loading per-directory configuration via `includeIf` directives in `~/.gitconfig`. The top-level config defines a default identity and then conditionally pulls in additional config files based on where the repo lives on disk:

```ini
# ~/.gitconfig
[user]
    name = Default Name
    email = default@example.com

[includeIf "gitdir:~/gh-acc/org1/"]
    path = ~/.gitconfig-org1

[includeIf "gitdir:~/gh-acc/org2/"]
    path = ~/.gitconfig-org2
```

The included files override the default identity with whatever's appropriate for that tree:

```ini
# ~/.gitconfig-org1
[user]
    name = Identity One
    email = identity1@org1.example.com

[core]
    sshCommand = ssh -i ~/.ssh/id_ed25519_org1 -o IdentitiesOnly=yes
```

```ini
# ~/.gitconfig-org2
[user]
    name = Identity Two
    email = identity2@org2.example.com

[core]
    sshCommand = ssh -i ~/.ssh/id_ed25519_org2 -o IdentitiesOnly=yes
```

To verify the right identity is loading inside a repo:

```bash
cd ~/gh-acc/org1/some-repo
git config --show-origin user.email
# /Users/you/.gitconfig-org1   identity1@org1.example.com
```

`--show-origin` is the part that matters — it tells you which file the value came from. If the default identity is showing up where you expect the per-tree one, the `includeIf` match isn't firing.

### Gotchas with `includeIf gitdir:`

A few things that bit me when I first set this up:

**The trailing slash is mandatory for directory matching.** `gitdir:~/gh-acc/org1` (no slash) is a glob pattern, not a directory match — it will also match `~/gh-acc/org10/`, `~/gh-acc/org1-archive/`, anything that starts with that prefix. Always end with `/` to constrain to a directory.

**`gitdir:` matches the path to the `.git` directory**, not the working tree. With a normal clone this distinction doesn't matter, but with `git worktree` or symlinked checkouts it can. Use `git config --show-origin user.email` from inside a representative repo to confirm.

**The `~` expansion works**, but earlier versions of git required `~/` to mean home explicitly. If you're on git older than 2.36, double-check by using the absolute path (`/Users/you/gh-acc/org1/`) instead.

**`gitdir:` is case-sensitive on Linux and case-insensitive on macOS by default.** Use `gitdir/i:` to force case-insensitive matching if you need it portable.

### One SSH key per identity

GitHub doesn't allow the same SSH public key to be registered on more than one account at a time, so each identity needs its own keypair:

```bash
ssh-keygen -t ed25519 -C "identity1@org1.example.com" -f ~/.ssh/id_ed25519_org1
ssh-keygen -t ed25519 -C "identity2@org2.example.com" -f ~/.ssh/id_ed25519_org2
```

Upload each public key under **Settings → SSH and GPG keys → New SSH key** while signed in as the matching identity.

The `core.sshCommand` line in each per-tree config tells git which private key to use when pushing or fetching. The `-o IdentitiesOnly=yes` flag is the important bit: without it, ssh will try every key loaded in your agent in turn, and GitHub will accept the *first* one that matches a known user — which may not be the identity you wanted. With `IdentitiesOnly=yes`, ssh uses only the key you named and fails fast otherwise.

An alternative is to define host aliases in `~/.ssh/config` and clone with them:

```
# ~/.ssh/config
Host github.com-org1
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_org1
    IdentitiesOnly yes

Host github.com-org2
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_org2
    IdentitiesOnly yes
```

```bash
git clone git@github.com-org1:org1/some-repo.git
```

I find `core.sshCommand` easier in practice because the alias-host approach requires every clone URL to use the right alias — easy to forget when copy-pasting URLs from GitHub.

### Or: HTTPS + credential helper

If you'd rather keep HTTPS clone URLs and let macOS Keychain (or `git-credential-manager` on other platforms) store your credentials, that works too. I actually use this path day-to-day. It's slightly less clean than SSH for multi-account but it's what most people already have set up. Two things to know:

**The "password" isn't your password.** GitHub disabled password authentication for git operations in August 2021. When `git push` over HTTPS prompts you for a password, it's asking for a **Personal Access Token (PAT)**. Pasting your GitHub login password will silently fail with an authentication error and you'll spend ten minutes wondering why. Use the same PAT you generated for `gh` (or a separate one — either works), as long as it has the `repo` scope for classic tokens, or **Contents: Read & write** for fine-grained tokens.

**Credential helpers key by host, not directory.** By default `osxkeychain` will store one credential for `github.com` and hand the same one back to both trees — exactly the multi-account problem we're trying to avoid. The fix is to specify the username in the org specific configuration

```ini
# ~/.gitconfig-org1
[user]
    name = Identity One
    email = identity1@org1.example.com

[credential "https://github.com"]
  username = identity1
```

```ini
# ~/.gitconfig-org2
[user]
    name = Identity Two
    email = identity2@org2.example.com

[credential "https://github.com"]
  username = identity2
```

Now any clone, fetch, or push of a `https://github.com/...` uses the username from the org config. Git prompts for the password (PAT) the first time, osxkeychain stores it keyed by the (host, user) pair, and the same setup runs independently in `org2` for `identity2`. The `core.sshCommand` line isn't needed in this mode — drop it from the per-tree config if you're going HTTPS-only.

First push in a fresh tree:

```bash
cd ~/gh-acc/org1/some-repo
git push
# Username for 'https://github.com': identity1   (auto-filled by the rewrite)
# Password for 'https://identity1@github.com': ← paste the PAT here
```

After that, subsequent pushes are silent. To inspect what got stored, open **Keychain Access → Login → search "github.com"**; each identity should be a separate row.

To rotate a token later, delete the matching keychain entry and let git re-prompt on the next push.

The reason SSH is theoretically cleaner is that it sidesteps the host-keying problem entirely (the key is bound to a specific identity at the SSH layer), and there's no token rotation step. But HTTPS works fine once the rewrite is in place, and corporate networks sometimes block port 22 for SSH while permitting 443 for HTTPS — so HTTPS is the more portable choice if you switch between networks.

That's the git side. The result is that a clone, commit, or push from anywhere under `~/gh-acc/org1/` runs under `identity1`'s name, email, and credentials — whether those credentials are an SSH key or a keychain-stored PAT — and the same goes for `~/gh-acc/org2/` and `identity2`. No per-shell switching required.

Now for `gh`.

## How gh authenticates

`gh` reads its credentials from one of two places, in priority order:

1. The `GH_TOKEN` environment variable, if set.
2. The macOS keychain (or equivalent secrets store), populated by `gh auth login`.

Critically, if `GH_TOKEN` is set, it wins over everything else *and* it suppresses `gh auth login` entirely:

```
$ gh auth login
? Where do you use GitHub? GitHub.com
The value of the GH_TOKEN environment variable is being used for authentication.
To have GitHub CLI store credentials instead, first clear the value from the environment.
```

This is by design — `GH_TOKEN` is the CI escape hatch — but it means if you have `GH_TOKEN` exported globally from `.zprofile` or `.zshrc`, the keychain flow is dead and you're stuck with a single token everywhere.

The fix: make `GH_TOKEN` get exported *conditionally*, based on which directory you're in.

## Token storage

I keep one file per account, with tight permissions:

```bash
mkdir -p ~/.config/gh-tokens
chmod 700 ~/.config/gh-tokens

# Personal access tokens generated at https://github.com/settings/tokens
# (or /settings/personal-access-tokens for fine-grained tokens), one per account,
# while signed in as the matching identity.
echo "ghp_xxxxxxxxxxxxx" > ~/.config/gh-tokens/org1
echo "ghp_yyyyyyyyyyyyy" > ~/.config/gh-tokens/org2
chmod 600 ~/.config/gh-tokens/*
```

These never end up in shell init or in any repo. They sit in their own directory in `~/.config/`, which is also where `gh` itself stores its config — feels at home.

## .envrc per tree

[direnv](https://direnv.net/) reads a `.envrc` file when you `cd` into a directory and exports whatever it sets, for as long as you remain inside that subtree. Leave the subtree, the exports get unloaded.

`~/gh-acc/org1/.envrc`:

```bash
TOKEN_FILE="${HOME}/.config/gh-tokens/org1"

if [ -r "${TOKEN_FILE}" ]; then
    export GH_TOKEN="$(cat "${TOKEN_FILE}")"
else
    echo "direnv: ${TOKEN_FILE} not found — gh will be unauthenticated under this tree." >&2
    unset GH_TOKEN
fi
```

`~/gh-acc/org2/.envrc`: same shape, different filename.

The `.envrc` goes at the *parent* of all my clones for that account, not inside each clone. That way every repo I add later under `~/gh-acc/org1/` inherits the right token without further setup.

Then approve each one once:

```bash
cd ~/gh-acc/org1 && direnv allow
cd ~/gh-acc/org2 && direnv allow
```

## Install the hook

This is the step that almost no one mentions in the screenshots, and it's the one that bit me. direnv has two completely separate pieces: the binary (which Homebrew installs and `direnv status` invokes directly), and the shell hook (which actually triggers loading on `cd`). The binary is useless on its own.

```bash
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc
exec zsh -l
```

`brew shellenv` does *not* do this for you — that's just `PATH` setup. They're complementary lines.

After the hook is installed, every `cd` into one of the trees prints:

```
$ cd ~/gh-acc/org2
direnv: loading ~/gh-acc/org2/.envrc
direnv: export +GH_TOKEN
```

That `export +GH_TOKEN` line is the receipt that it actually worked.

## Verifying

```bash
cd ~
echo "${GH_TOKEN:-unset}"       # unset

cd ~/gh-acc/org2
gh api user --jq .login         # identity2

cd ~/gh-acc/org1
gh api user --jq .login         # identity1
```

Switching directories switches identities, no extra commands.

## Gotchas I hit

A handful of things tripped me up in roughly the order I hit them.

**GH_TOKEN in your shell init blocks `gh auth login` silently.** If you've ever pasted a token into `.zprofile` or `.zshrc` and forgotten about it, `gh auth login` will refuse to do anything until you remove it. The error message (quoted above) is clear once you read it, but if you're skimming output it looks like a normal banner.

**zsh doesn't treat `#` as a comment in interactive shells by default.** I was copy-pasting verify commands like:

```
gh api user --jq .login            # → expected login
```

and getting:

```
accepts 1 arg(s), received 5
```

That's because zsh passed `#`, `→`, `expected`, `login` as positional arguments to `gh api`. Fix is either to drop the trailing comments or to `setopt interactive_comments` in `~/.zshrc`. I did the latter so I stop tripping on this.

**`Found RC allowed 0` in `direnv status` means the file IS allowed.** Counterintuitive given that `0` reads like a boolean false. In direnv's status output, `0 = Allowed`, `1 = NotAllowed`, `2 = Denied`. So when you're debugging and see:

```
No .envrc or .env loaded
Found RC path /Users/.../gh-acc/org2/.envrc
Found RC allowed 0
```

…the `.envrc` is fine. The problem is somewhere else.

**That "somewhere else" is usually the missing hook.** If you installed direnv but never added `eval "$(direnv hook zsh)"` to `~/.zshrc`, then `direnv status` works (it shells out to the binary directly), but `cd` never triggers loading. You'll see `Found RC allowed 0` (allowed!) and `No .envrc or .env loaded` (not loaded!) at the same time, which is exactly the failure mode I was staring at. The fix is the one line above plus `exec zsh -l`.

**The hook is per-shell, not per-directory.** Once you've installed it once in `~/.zshrc`, every new shell session everywhere inherits it. There's no need to repeat any setup as you add new `.envrc` files or new account trees later.

## Why I wanted this

I need to use the gh cli to pull data from github for different project for example one of the things on my list is triaging the Dependabot alerts that have accumulated. Reading those alerts via `gh api /repos/owner/repo/dependabot/alerts` requires authenticating as the repo user — one specific identity. Before this setup, every Dependabot-alert query meant `gh auth switch`, fetch, `gh auth switch` back, because my shells were typically already authenticated under a different identity. With direnv, I just `cd` into the project and the right token is already in the environment.

Worth the half-hour setup.

## Future things I'd like to do

A few extensions to this setup that I haven't gotten to yet but that slot in cleanly:

**Commit signing per identity.** Every commit you make inside a tree could be cryptographically signed with a key tied to that identity. GitHub displays a "Verified" badge next to signed commits, and some orgs require signing on protected branches. SSH-based signing (introduced in git 2.34) is the easier path because it reuses the SSH keys I'm already using for push auth. The per-tree configs grow these lines:

```ini
# ~/.gitconfig-org1
[user]
    signingkey = ~/.ssh/id_ed25519_org1.pub

[commit]
    gpgsign = true

[gpg]
    format = ssh
```

…and the same public key gets registered with GitHub a second time, under the same SSH-keys page, with **Key type = Signing Key** instead of Authentication Key. When I finally set this up I'll update the post.

**Per-tree shell prompt.** A small `direnv` addition that prepends the current identity (or org name) to the shell prompt while you're inside a tree — a visual reminder of which account is active. Useful when you have multiple terminals open and they look identical.

**A `direnv` helper for the credential helper.** If you ever do need HTTPS auth (some workflows can't avoid it), you can point `credential.helper` at a per-tree token file the same way we point `GH_TOKEN` at `~/.config/gh-tokens/orgN`. The mechanics are similar to the gh setup but using `git credential-store` or a tiny custom helper script.

---

## Addendum, August 2026: what broke after three months

Three things went wrong in the months after I wrote this, all of them in the HTTPS half of the
setup. Two were self-inflicted and one is a GitHub limitation with no clean workaround. Each cost
me more time than it should have, so they go here rather than in a new post.

### `gh auth setup-git` silently takes over the credential chain

**Symptom:** VS Code started prompting for a password on every push, in every tree, while the same
pushes from the terminal kept working.

**Cause:** at some point I ran `gh auth setup-git` — or, more likely, accepted a prompt offering to
"configure git authentication" for me. It writes this into `~/.gitconfig`:

```ini
[credential "https://github.com"]
	helper =
	helper = !/opt/homebrew/bin/gh auth git-credential
```

That first, empty `helper =` is the destructive line. An empty value **resets the helper list**,
wiping the `osxkeychain` helper inherited from the system gitconfig at
`/Library/Developer/CommandLineTools/usr/share/git-core/gitconfig`. What's left is `gh` — and `gh`
has exactly one active account per host. So every tree got the same token, regardless of
`includeIf`, regardless of the `credential.https://github.com.username` pin from the per-tree
config. Mine was my personal account, which has no access to my employer's org, so GitHub answered
403 and git fell back to prompting.

Two commands diagnose this. The first shows the whole chain and which file each piece came from:

```bash
git config --list --show-origin --show-scope | grep -i -e credential -e helper
```

The second asks git what it would actually hand to GitHub:

```bash
printf 'protocol=https\nhost=github.com\n\n' | git credential fill
```

Read the token prefix in the output. `gho_` means you're going through `gh`'s OAuth login, not the
keychain. `ghp_` is a classic PAT, `github_pat_` a fine-grained one.

The fix, and the thing never to re-run:

```bash
git config --global --unset-all credential.https://github.com.helper
git config --global --unset-all credential.https://gist.github.com.helper
```

`gh auth setup-git` is aimed at people who don't already have a credential story. If you've built
one deliberately, it is strictly destructive — and nothing warns you, because the terminal keeps
working for as long as `gh`'s active account happens to be the right one.

### Keep git credentials out of the environment — VS Code never sees it

While debugging the above I went down a long detour building a custom credential helper that read
`$GH_TOKEN` and a per-tree token file. It worked in the terminal and did nothing at all in VS Code,
which is exactly the wrong way round: the terminal was never the problem.

The reason is worth stating plainly. **direnv only affects shells.** VS Code's git integration runs
in the extension-host process, which inherits whatever environment VS Code was launched with — and
if you start it from the Dock or Spotlight, that's essentially nothing. No `.envrc` is ever
evaluated for it. So any git auth scheme built on environment variables works in a direnv-hooked
shell and fails in the editor, in Fork, in Tower, and in anything else launched from the GUI.

The keychain path has no such problem: it's git config plus a helper binary, no environment
involved. Which makes VS Code a useful canary — if a push works in the terminal but prompts in the
editor, something env-dependent has crept into the credential path.

This also retracts the last bullet in "Future things I'd like to do" below: wiring
`credential.helper` to a per-tree token file the way `GH_TOKEN` is wired is a bad idea for exactly
this reason. Don't do it. The keychain already gives you per-tree separation, and it gives it to
every client rather than only to shells with the hook installed.

### Key credentials per repo, not just per identity: `useHttpPath`

The post above says credential helpers key by host, and that pinning `username` per tree upgrades
that to (host, user). There's a third level I've since started relying on:

```ini
# ~/.gitconfig-org1
[credential]
	useHttpPath = true
```

With this on, git passes the repo path down to the helper, so the keychain key becomes
(host, user, path) and **each repo gets its own stored token**. I need this because my fine-grained
tokens are deliberately scoped to a couple of repos each; without it, one token per identity has to
cover every repo in the tree, which is the opposite of least privilege. First push in each repo
prompts once, then goes quiet.

The rotation command has to include the path, or it won't match what got stored:

```bash
printf 'protocol=https\nhost=github.com\npath=owner/repo.git\n\n' \
  | git credential-osxkeychain erase
```

One caveat: `gh` ignores `useHttpPath` entirely. If `gh` is anywhere in your credential chain (see
the first section), this setting silently does nothing.

### ProjectsV2 still needs a classic PAT

This one isn't fixable, only workable-around. The scripts from
[Story points on GitHub Projects from git history]({% post_url 2026-06-27-StoryPointsOnGithubProjectsFromGitHistory %})
started failing with:

```
GraphQL: Resource not accessible by personal access token (addProjectV2ItemById)
```

The tell is that **reads succeed and writes fail**. `gh project list`, `field-list`, and `item-list`
all worked; `item-add` and `item-edit` were rejected. Fine-grained PATs can read ProjectsV2 but not
write it — you need a classic PAT with the `project` scope.

The obvious move is to add a second token to the tree's `.envrc`. That's wrong twice over. It makes
the token ambient for everything in the tree, and a classic PAT carrying only `project` has no
`repo` scope, so it breaks the `gh issue view` and `gh issue list` calls in the same scripts. Giving
it `repo` as well defeats the entire point of the fine-grained token.

So inject it per command instead. In the shared library the scripts already source:

```bash
# ProjectsV2 mutations require a CLASSIC PAT with the `project` scope; fine-grained
# PATs can read projects but cannot write them. Keep that token out of the ambient
# environment: inject it only for `gh project` calls, so `gh issue`, git, and
# everything else keep using the repo-scoped token direnv exports for this tree.
PROJECT_TOKEN_FILE="${PROJECT_TOKEN_FILE:-$HOME/.config/gh-tokens/acct-projects}"

_project_token() {
  if [ ! -r "$PROJECT_TOKEN_FILE" ]; then
    echo "ERROR: no ProjectsV2 token at $PROJECT_TOKEN_FILE" >&2
    return 1
  fi
  tr -d '\r\n' < "$PROJECT_TOKEN_FILE"
}

# Intercept `gh project ...` only; every other gh invocation passes through
# untouched. `command gh` avoids recursing into this function.
gh() {
  if [ "${1:-}" = project ]; then
    local _tok
    _tok="$(_project_token)" || return 1
    GH_TOKEN="$_tok" GITHUB_TOKEN="$_tok" command gh "$@"
  else
    command gh "$@"
  fi
}
```

Shell functions are inherited by subshells, so this covers `$(gh project item-list ...)` inside
command substitutions too, and every script that sources the library gets it for free with no
call-site changes.

Classic `project` is read+write across *all* your projects, which is coarser than I'd like and
there's no finer classic scope. But with no `repo` scope the token can't read private code or push
anything, which is the containment that actually matters if it leaks.

### The one command worth remembering

Of everything in this post, `git credential fill` is the piece I now reach for first. It answers
"what would git actually send for this repo, and where did that come from" in one line, which is
the question underneath nearly every multi-account credential problem:

```bash
printf 'protocol=https\nhost=github.com\npath=owner/repo.git\n\n' | git credential fill
```
