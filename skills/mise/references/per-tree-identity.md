## Per-Tree Identity (git, gh, npm, Claude Code)

**Contents**

- [When to Use This](#when-to-use-this)
- [Layout](#layout)
- [Global Default](#global-default)
- [The gh Token Script](#the-gh-token-script)
- [Override Trees](#override-trees)
- [Two `_.source` Scripts in One Tree](#two-_source-scripts-in-one-tree)
- [Git Identity (`includeIf`)](#git-identity-includeif)
- [npm and pnpm Config per Tree](#npm-and-pnpm-config-per-tree)
- [Claude Code Config Dir per Tree](#claude-code-config-dir-per-tree)
- [Verify: an `identity` Task](#verify-an-identity-task)
- [AI Agent Shells Ignore the Tree After `cd`](#ai-agent-shells-ignore-the-tree-after-cd)
- [Checklist](#checklist)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

### When to Use This

Use this recipe when one machine holds work for more than one account, such as personal and work GitHub accounts, two npm registries, or two Claude Code logins, and each directory tree should use its own account automatically. mise sets the GitHub token, npm user config, and Claude Code config directory per tree. git's own `includeIf` handles commit identity, and an `identity` task checks that both agree.

Verified on mise 2026.10.25 with gh 2.103.3, git 2.56.0, npm 11.20.0, and pnpm 12.17.0.

### Layout

```text
~/.config/mise/config.toml          # default identity for every directory
~/.config/mise/scripts/gh-token.sh  # exports GH_TOKEN/GITHUB_TOKEN for $EXPECTED_GH_USER
~/.config/mise/tasks/identity       # reports and verifies the identity in effect
~/work/mise.toml                    # override tree: work account, own .npmrc, own Claude config
~/work/.npmrc
~/personal/mise.toml                # optional: restates the default and adds its own .npmrc
~/.gitconfig                        # includeIf rules: git's side of the same split
```

A project config wins over the global config, and a tree's `mise.toml` applies to every directory below it, so one file per tree is enough. Trust each tree once with `mise trust ~/work/mise.toml`. An untrusted config fails with `Config files in … are not trusted`.

### Global Default

```toml
# ~/.config/mise/config.toml
[tools]
gh = "latest"

[env]
# Declares the identity for every directory without its own override.
EXPECTED_GIT_EMAIL = "you@example.com"
EXPECTED_GH_USER = "personal-user"
# Source the token script once, here. tools = true defers it until gh is on PATH.
_.source = { path = "{{ env.HOME }}/.config/mise/scripts/gh-token.sh", tools = true, redact = true }
```

- **`EXPECTED_*`**: these are plain variables. The token script reads `EXPECTED_GH_USER`, and the `identity` task compares both against what git and gh actually use.
- **`tools = true`**: runs the script after mise has resolved tools, so `gh` is on `PATH`. It also runs after **every** plain `[env]` entry from **every** config file. The global script therefore sees an override tree's `EXPECTED_GH_USER` (verified), and trees only need to set the variable.
- **`redact = true`**: marks every variable the script exports as sensitive, so the token is masked in task output and listed by `mise env --redacted`.

### The gh Token Script

```bash
# ~/.config/mise/scripts/gh-token.sh — sourced by mise (bash --norc); the shebang is ignored.
# Exports the token of $EXPECTED_GH_USER from gh's keyring. Never writes it to disk.
# Non-fatal: if that account isn't logged in, nothing is exported and gh falls back to
# its active account (`mise run identity` shows which account is in effect).
if [ -n "${EXPECTED_GH_USER:-}" ]; then
  _gh_tok=$(gh auth token --user "$EXPECTED_GH_USER" 2>/dev/null)
else
  _gh_tok=$(gh auth token 2>/dev/null)
fi
if [ -n "${_gh_tok:-}" ]; then
  export GH_TOKEN="$_gh_tok"      # gh itself uses this account
  export GITHUB_TOKEN="$_gh_tok"  # .npmrc, terraform's github provider, and similar read this one
fi
unset _gh_tok
```

- **Why a token:** `gh auth switch` changes one active account shared by every shell. A `GH_TOKEN` in the environment takes precedence over the active account, so each tree keeps its own account and nothing needs switching.
- **Inherited tokens:** `gh auth token --user X` returns X's stored token even when `GH_TOKEN` is already set (verified). The script still works in a process that inherited another tree's token.
- **Re-authenticating:** while either variable is set, `gh auth login` and `gh auth switch` refuse to run ("The value of the GH_TOKEN environment variable is being used…"). `GITHUB_TOKEN` alone also blocks them, so clear both: `env -u GH_TOKEN -u GITHUB_TOKEN gh auth login`.
- **Cost:** the script runs every time mise computes the environment, so keep it fast. See [cost](#ai-agent-shells-ignore-the-tree-after-cd).

### Override Trees

```toml
# ~/work/mise.toml
[env]
EXPECTED_GIT_EMAIL = "you@work.example.com"
EXPECTED_GH_USER = "work-user"
NPM_CONFIG_USERCONFIG = "{{ config_root }}/.npmrc"
CLAUDE_CONFIG_DIR = "{{ env.HOME }}/.claude-work"
```

No `_.source` is needed here. The global script is deferred by `tools = true`, so it already reads `work-user`. If the tree sources `gh-token.sh` again, the script runs twice per environment computation. That makes two `gh auth token` calls, doubling the script's cost on every shimmed call. Source it in a tree only when that tree must work without the global config, and then not globally as well.

A tree that matches the default can still restate it (`~/personal/mise.toml`). That's useful when the tree needs its own `.npmrc`, or when you want `identity` to show the expectation explicitly.

### Two `_.source` Scripts in One Tree

A tree sometimes needs a second, different script, such as one that reads a publish token from the OS keychain. Two `_.source` keys in one `[env]` table are invalid TOML:

```text
× Invalid TOML in config file
 6 │ _.source = { path = "b.sh", tools = true, redact = true }
   ·   ───┬──
   ·      ╰── duplicate key
```

mise then rejects the **whole file**: `mise env` exits 1, and every shimmed command in that tree fails. Two forms work, and both keep per-script options:

```toml
# A: one [env] table; an array whose items are directive tables
[env]
PUBLISH_KEYCHAIN_SERVICE = "work-publish"
_.source = [
  { path = "{{ env.HOME }}/.config/mise/scripts/publish-creds.sh", redact = true },
  { path = "{{ env.HOME }}/.config/mise/scripts/other.sh", tools = true, redact = true },
]
```

```toml
# B: [[env]] blocks, resolved in order (the whole file must use [[env]])
[[env]]
PUBLISH_KEYCHAIN_SERVICE = "work-publish"
_.source = { path = "{{ env.HOME }}/.config/mise/scripts/publish-creds.sh", redact = true }

[[env]]
_.source = { path = "{{ env.HOME }}/.config/mise/scripts/other.sh", tools = true, redact = true }
```

Forms that look similar but behave differently:

| Form | Result |
|------|--------|
| `_.source = ["a.sh", "b.sh"]` | Both run in order, but plain strings carry **no options**: no `tools`, no `redact` |
| `_.source = { path = ["a.sh", "b.sh"], tools = true }` | The same options apply to **every** script |
| `[env]` and `[[env]]` in the same file | Parse error at `[[env]]` (`duplicate key`). Convert the whole file |
| `_.source` written below a `[tasks.*]` header | Belongs to that task. Uncommented, it is a parse error (`unknown field '_'`). Commented out, it silently does nothing. Keep every `[env]`/`[[env]]` block above the first `[tasks.*]` table |

**Order:** plain variables and scripts without `tools = true` resolve in the order written. A script only sees variables above it, whether earlier in the table or in an earlier `[[env]]` block (verified). Scripts with `tools = true` run after every plain entry, so they see later ones too. Still declare the variable a script reads above it, so the file reads correctly and survives dropping `tools = true`.

### Git Identity (`includeIf`)

Commit identity belongs in git config, not mise. Every git client honors it, including GUI clients and a system `git` that no mise shim wraps.

```ini
# ~/.gitconfig
[user]
	name = Your Name
	email = you@example.com
# Includes apply top to bottom and the last match wins:
# tree -> work remotes cloned elsewhere -> personal exceptions inside the work tree.
[includeIf "gitdir:~/work/"]
	path = ~/.gitconfig-work
[includeIf "hasconfig:remote.*.url:git@github.com:work-org/**"]
	path = ~/.gitconfig-work
[includeIf "hasconfig:remote.*.url:git@github.com:personal-user/**"]
	path = ~/.gitconfig-personal
```

- **`gitdir:`** with a trailing `/` matches every repository below the directory, including a fresh `git init` before any remote exists.
- **`gitdir:` only applies inside a repository.** In a plain directory under `~/work/`, `git config user.email` returns the default (verified). Don't treat that as the identity a new repo there would get.
- **`hasconfig:remote.*.url:`** matches by remote URL, so it catches work repos cloned outside the tree. It needs a remote.
- Put `user.email`, `user.signingkey`, and signing settings in the included files, so each identity signs with its own key.

### npm and pnpm Config per Tree

`NPM_CONFIG_USERCONFIG` replaces `~/.npmrc` as the **user** config. npm 11.20.0 and pnpm 12.17.0 both read the registry from the file it names (verified). Other clients may not, so check yours.

```ini
# ~/work/.npmrc. Keep secrets out of it: npm expands ${VAR} from the environment.
registry=https://npm.work.example.com/
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

With the token script, `${GITHUB_TOKEN}` resolves to the tree's GitHub account, and no credential is written to disk. A project-level `.npmrc` inside a repo still applies on top.

### Claude Code Config Dir per Tree

`CLAUDE_CONFIG_DIR` (default `~/.claude`) moves Claude Code's settings, session history, and plugins to another directory, giving a tree its own login. Run `claude` in the tree once and use `/login`.

- **Read at launch.** It follows the directory where `claude` **starts**. A session started elsewhere keeps its own config dir after a `cd`.
- **Set it in the shell, not in project settings.** Claude Code ignores it in project and local settings, so mise `[env]` is the right place. Write an absolute path; `{{ env.HOME }}` renders one.
- **`claude` is not a mise tool,** so no shim applies `[env]` to it. Start it from a fully activated shell in the tree, or with `mise x -- claude` from a shims-only shell. `mise x` applies the current directory's `[env]` to any command.

### Verify: an `identity` Task

A global file task reports what git and gh actually use and compares it with the tree's `EXPECTED_*` values:

```bash
#!/usr/bin/env bash
#MISE description="Report the git and gh identity in effect for the current directory"
#MISE dir="{{cwd}}"
#USAGE flag "-s --strict" help="Exit non-zero when the identity in effect differs from the tree's EXPECTED_* values"
#USAGE flag "-n --no-net" help="Skip the GitHub API call; report local configuration only"
set -uo pipefail

want_email="${EXPECTED_GIT_EMAIL:-}"
want_gh="${EXPECTED_GH_USER:-}"
mismatch=0

echo "cwd         $PWD"
echo "declares    ${want_email:-<no EXPECTED_GIT_EMAIL>} / ${want_gh:-<no EXPECTED_GH_USER>}"

# git: includeIf "gitdir:" rules are only evaluated inside a repository
if git rev-parse --git-dir >/dev/null 2>&1; then
  email=$(git config user.email || true)
  echo "git email   ${email:-<unset>}  ($(git config --show-origin user.email | cut -f1))"
  if [ -n "$want_email" ] && [ "$email" != "$want_email" ]; then
    echo "  MISMATCH  expected $want_email"
    mismatch=1
  fi
else
  echo "git email   skipped: not a repository (gitdir: rules don't apply here)"
fi

# gh: a GH_TOKEN in the environment wins over the keyring's active account
if [ -n "${GH_TOKEN:-}" ]; then
  echo "gh token    GH_TOKEN (pinned by mise)"
else
  echo "gh token    keyring active account (shared by every shell)"
fi
if [ "${usage_no_net:-false}" != "true" ]; then
  login=$(gh api user --jq .login 2>/dev/null || true)
  echo "gh account  ${login:-<gh api call failed>}"
  if [ -n "$want_gh" ] && [ "$login" != "$want_gh" ]; then
    echo "  MISMATCH  expected $want_gh"
    mismatch=1
  fi
fi

if [ "$mismatch" -eq 0 ]; then
  echo "identity matches this tree"
else
  echo "identity does NOT match this tree"
  [ "${usage_strict:-false}" = "true" ] && exit 1
fi
exit 0
```

Save it as `~/.config/mise/tasks/identity`, make it executable, and run `mise run identity` (add `--strict` in scripts and hooks).

- **`dir="{{cwd}}"` is required.** A global file task otherwise runs in `~` even though its `[env]` comes from the directory you ran it in (verified). Without it, the git checks would inspect the wrong directory.
- A personal repo inside the work tree, kept personal by an `includeIf` exception, reports a git mismatch by design. Add a `mise.toml` in that repo that restates the personal `EXPECTED_*` values.

### AI Agent Shells Ignore the Tree After `cd`

**Symptom:** inside Claude Code, `gh` uses the wrong account in a pinned tree, while `mise run identity` in a normal terminal reports a match.

**Cause:** at session start, Claude Code sources `~/.zshrc` (or `~/.bashrc`) once. It saves the resulting functions, aliases, options, and `PATH` in a snapshot and replays it before every command, and no prompt hooks run afterward. Full `mise activate` produces two problems:

- **The env is frozen at the start directory.** Variables come from the start directory's environment, and a later `cd ~/work` never runs `hook-env`, so the tree's `mise.toml` never applies.
- **Shims are bypassed.** The snapshot's `PATH` lists tool install dirs **ahead of** the shims dir, so even the shims don't run.

The same steps reproduced in isolation: a command after `cd` into an override tree still saw the start tree's token.

**Fix:** use shims-only activation when `CLAUDECODE` is set (Claude Code sets it to `1` in the shells it spawns):

```zsh
# ~/.zshrc
if [[ -n ${CLAUDECODE:-} ]]; then
  eval "$(mise activate zsh --shims)"   # only prepends the shims dir to PATH
else
  eval "$(mise activate zsh)"
fi
```

Each shim resolves the tool **and** the `[env]` of its current directory for that one process. `gh` run in `~/work` gets the work token, and in `~` the default (verified). Use the same pattern with `mise activate bash --shims` in `~/.bashrc`.

- **Only shimmed tools see the values.** `echo $GH_TOKEN` in the agent shell prints nothing. mise tools and `mise run`/`mise x` get the tree's env, but non-mise binaries (system `git`, `curl`) don't. That is why git identity lives in `includeIf`.
- **Cost:** every shimmed call recomputes the environment and runs every `_.source` script. A bare shim adds tens of milliseconds. A script that shells out, such as `gh auth token`, typically adds more than 100 ms per call, and a second copy of it adds the same again. Source each script once, and time a shimmed command (`time gh --version`) after changing `[env]`.
- **`env_cache` does not help.** The cache only works with a session key that full `mise activate` or `mise exec` puts in the environment, and `activate --shims` doesn't. With `env_cache` and `env_cache_redacted` both on, three shim calls still ran the script three times (verified).
- **Restart the agent session** after editing the rc file. The snapshot is taken once at start, and a session started before the edit keeps full activation.
- **IDE terminals.** Claude Code's docs say IDE extensions also set `CLAUDECODE` in their integrated terminals, so those terminals get shims-only activation too. Start `claude` there with `mise x -- claude` so it picks up the tree's `CLAUDE_CONFIG_DIR`.
- **Other agents and IDEs** that set up a shell once and reuse it may hit the same problem. If commands ignore a tree's `[env]` after `cd`, check whether the tool snapshots its shell.

### Checklist

- Define `EXPECTED_*` in the global config and in each override tree, and source `gh-token.sh` once, globally, with `tools = true, redact = true`
- Run `mise trust` on each tree's `mise.toml`
- Give a tree that needs two scripts one array of directive tables, or convert the file to `[[env]]`. Never write two `_.source` keys in one table
- Keep `[env]`/`[[env]]` above the first `[tasks.*]` header
- Route commit identity with `includeIf "gitdir:…"` plus `hasconfig:remote.*.url:` rules, with personal exceptions last
- Use `NPM_CONFIG_USERCONFIG = "{{ config_root }}/.npmrc"` for a per-tree npm/pnpm user config, with tokens referenced as `${GITHUB_TOKEN}`
- Use `CLAUDE_CONFIG_DIR` for a per-tree Claude Code login. It is read when `claude` starts
- Use `mise activate <shell> --shims` when `CLAUDECODE` is set, then restart agent sessions
- Check any directory with `mise run identity --strict`
