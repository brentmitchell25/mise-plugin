## Machine Bootstrap (Developer Setup)

**Contents**

- [[bootstrap] Configuration](#bootstrap-configuration)
- [Remote Bootstrap over SSH ([bootstrap.remote])](#remote-bootstrap-over-ssh-bootstrapremote)
- [Declarative Dotfiles ([dotfiles])](#declarative-dotfiles-dotfiles)
- [Dotfile Groups ([dotfile_groups])](#dotfile-groups-dotfile_groups)
- [Dotfiles History (mise dot / [history])](#dotfiles-history-mise-dot--history)

> The usage-field rule in `SKILL.md` applies to every task example in this file.

`mise bootstrap` (alias **`bs`**) provisions an entire machine from declarative config. **Stable since v2026.7.4** (no longer requires `MISE_EXPERIMENTAL`). Its declared effect is **destructive**.

```bash
mise bootstrap                    # run the full setup
mise bootstrap -n                 # --dry-run: preview without changing anything
mise bootstrap -y                 # --yes: skip confirmation prompts
mise bootstrap --update           # refresh package-manager metadata (and repos) first; without
                                   # --from/--adopt it also fast-forwards a git-checkout ~/.config/mise
                                   # and re-runs (2026.11.0; --dry-run prints `git pull --ff-only`;
                                   # a diverged checkout stops the run)
mise bootstrap --force-dotfiles   # overwrite conflicting whole-file dotfiles
mise bootstrap --prompt-secrets   # prompt for [bootstrap.secrets] inputs
mise bootstrap --only packages,dotfiles
mise bootstrap --skip macos-defaults
mise bootstrap --skip-dirty        # 2026.8.13+ skip repos with uncommitted changes
mise bootstrap --adopt             # adopt an existing setup repo (`--from-git` was REMOVED in 2026.10.0)
mise bootstrap --adopt 'owner/repo?ref=work'   # 2026.10.10: adopt a branch (branches only; URLs with
                                   # embedded credentials are rejected)
mise bootstrap --upgrade           # 2026.10.11: also upgrade packages/tools
mise bootstrap --prune-dotfiles    # 2026.10.15: remove orphaned dotfile-group files
mise bootstrap --prompt-vars       # 2026.10.6: ask for unanswered [vars] prompts
mise bootstrap --take-remote-all   # resolve all dotfile-history conflicts with the remote side
mise bootstrap --from <url> -E workstation --save-env   # 2026.11.1: write the -E / MISE_ENV selection
                                   # (env_sets names allowed) into miserc.local.toml
mise bootstrap --from https://github.com/you/dotfiles.git --from-dir ~/.dotfiles --config-dir .config/mise
                                   # 2026.10.21: link ~/.config/mise to a dir inside the checkout
                                   # (requires --from; path stays inside the checkout; the existing
                                   # config dir must be missing, empty, or already that link)
mise bootstrap --from 'git::https://github.com/me/dotfiles.git?ref=v1'   # 2026.9.18: ?ref= pins a
                                   # branch/tag/commit; `git::` optional; --update re-resolves the ref
mise bootstrap unapply ssh --dry-run   # 2026.9.13: remove what a deselected module set up (destructive)
mise bootstrap plan [--json] [--detailed-exitcode]  # 0 = no changes, 2 = changes, 1 = failure
mise bootstrap status [--json] [--missing]          # non-zero exit when out of sync
mise bootstrap remote [TARGET]…   # apply config to inventory hosts / SSH destinations
```

**Declarative resource model (2026.8.2+).** Bootstrap converged into a Terraform-style plan/apply system. Resources have stable identities and a dependency graph, and are validated for duplicates, missing dependencies, and cycles. Each resource type also has its own `apply`/`status` pair that converges only when something actually differs:

```bash
mise bootstrap plan --json --detailed-exitcode   # 0 = no changes, 2 = changes, 1 = error
mise bootstrap files apply
mise bootstrap accounts status
mise bootstrap services apply
mise bootstrap compose apply
mise bootstrap firewall apply
mise bootstrap secrets status      # reports availability without revealing values
mise bootstrap packages prune --manager brew-cask           # lists what --force would also remove
mise bootstrap packages prune --manager brew-cask --force   # 2026.10.21: also self-updating casks, casks with
                                   # install steps, and casks installed before prune metadata existed
```

Subcommands: `accounts`, `compose`, `files`, `firewall`, `linux` (`systemd-units`), `macos` (`defaults`, `launchd-agents`), `mise-shell-activate`, `packages` (incl. `where`, `list` alias 2026.10.16), `plan`, `plugins`, `remote`, `repos`, `secrets`, `services` (incl. `remove`), `status` (alias `ls`), **`unapply`** (2026.9.13), `user`. `mise bootstrap dotfiles …` still works but is **hidden** since 2026.10.5 — use **`mise dot …`**. The old spellings `mise bootstrap launchd|systemd|macos-defaults` **warn** (since 2026.10.4, removed **2027.10.4**) — use `macos launchd-agents`, `linux systemd-units`, `macos defaults`.

**Modules and `mise bootstrap unapply` (2026.9.13).** A *module* is a configuration environment — `~/.config/mise/config.<name>.toml` (or `mise.<name>.toml` in a bootstrap project) — selected per machine via `~/.config/mise/miserc.local.toml` `env = ["ssh", "gpg"]` (replaces a shared `miserc.toml` list and stays local under `--adopt`) or `mise -E ssh,gpg bootstrap` (replaces the selection for one run) (per-host `mise_env` for remotes). Later envs in the list win for the same key. To retire one: remove it from `env`, **keep its file on disk**, then:

```bash
mise bootstrap unapply ssh --dry-run
mise bootstrap unapply ssh gpg --yes
```

Removal is planned from the current config *with vs. without* the named envs (not from run history): managed files, directories, user services, systemd user units, and dotfile entries/edits it contributed are removed, unless still declared elsewhere (an `absent` declaration does not protect). Changed targets are skipped unless `--force`; dirs are removed only if empty. Packages, repos, and Compose projects need separate cleanup (the output says how); system services are out of scope. `mise bootstrap plan --json` reports `origin.config` / `origin.environment` per resource.

`--only` / `--skip` are mutually exclusive, repeatable or comma-separated. Parts: `accounts`, `plugins`, `packages`, `files`, `services`, `firewall`, `compose`, `repos`, `dotfiles`, `mise-shell-activate`, `macos-defaults`, `macos-launchd-agents`, `linux-systemd-units`, `user`, `tools`, `task`, `final-hook`. The short aliases `shell`, `defaults`, `launchd`, `systemd` **warn** since 2026.10.4 (removed **2027.10.4**).

**Execution order (20 steps — `https://mise.jdx.dev/bootstrap.html#how-it-runs`):**

1. `accounts` — `[bootstrap.users]` / `[bootstrap.groups]` (Linux)
2. `plugins` — vfox plugins acting as package managers
3. `files` with `phase = "pre-packages"`
4. `packages` — `pre-packages` hook → `[bootstrap.apt.sources]` / `[bootstrap.dnf.repos]` → built-in-manager entries → `post-packages` hook
5. remaining `files` / `directories`
6. `services` (except user services with `requires_tools = true`)
7. `firewall` — `[bootstrap.linux.firewall]`
8. `compose` — Docker Compose projects
9. `repos` — `[bootstrap.repos]` git checkouts
10. `dotfiles` — `[dotfiles]` / `[dotfile_groups]` (same as `mise dot apply`)
11. `mise-shell-activate` — shell rc activation snippets
12. `macos-defaults` (incl. `[bootstrap.macos.login_items]`)
13. `macos-launchd-agents`
14. `linux-systemd-units`
15. `user` — `[bootstrap.user]` login shell
16. `tools` — `mise install`
17. plugin `packages` (if any package comes from a plugin manager, the `post-packages` hook runs **here** instead of after step 4)
18. user `services` with `requires_tools = true` (runs even with `--skip tools`)
19. `task` — `mise run bootstrap` (if defined)
20. `final-hook` — `[bootstrap.hooks.final]`

The secrets preflight also covers `[dotfiles]` `mode = "template"` entries. Bootstrap offers to install `git` and `ssh` before cloning on a fresh machine (2026.10.6). History checkpoints are recorded only when at least one file is tracked or history already exists.

Each phase is runnable on its own: `mise bootstrap packages apply`, `mise bootstrap macos defaults apply`, `mise bootstrap linux systemd-units apply`, `mise bootstrap repos update`, `mise bootstrap user apply`, etc. `mise bootstrap packages brew tap|untap` manages third-party Homebrew taps; `mise bootstrap packages import|prune|upgrade|use` syncs brew formulae; `mise bootstrap repos exec` runs a command across checkouts.

> System-package installs are gated by `[settings.system_packages]` (`sudo`, `managers`, **`elevate_with`** — since 2026.10.9 mise falls back from `sudo` to `run0` (systemd 256+) then `doas`; **global config only** since 2026.10.20 (a project value is ignored with a warning); `elevate_with = ""` means never elevate (2026.10.22, e.g. Termux); pin one with `elevate_with = "run0"` / `MISE_SYSTEM_PACKAGES_ELEVATE_WITH`; **`ostree`** — on Fedora Atomic/rpm-ostree hosts `"error"` (default) or `"layer"`, which uses `rpm-ostree install --idempotent --allow-inactive`, shows `pending reboot`, skips pins, and fails `upgrade` with a hint). The related `system_deps` setting (`prompt` default, `auto`, `warn`, `ignore`) controls how vfox `PLUGIN.systemDependencies` are surfaced — `prompt` falls back to `warn` non-interactively, and detection never fails an install.

### `[bootstrap]` Configuration

```toml
# ⚠️ config_roots is DEPRECATED (2026.9.4; hidden from help, removal 2027.3.3) — emits a warning.
# Migrate each bundle to a conf.d FOLDER fragment (~/.config/mise/conf.d/<name>/mise.toml, 2026.9.14),
# which is its own config root and keeps the module's files beside it.
[bootstrap]
config_roots = ["bundles/*"]
dotfile_groups = ["home", "zsh"]   # 2026.10.3: which [dotfile_groups] apply on THIS machine
                                   # (unset = all groups; a more-local list replaces others)

# System packages — managers: apk: apt: aur: brew: brew-cask: choco: dnf: flatpak: flatpak-user:
#   macos-app: mas: nix: pacman: scoop: snap: winget: zypper:
[bootstrap.packages]
"apt:build-essential" = "latest"
"apt:gcc:arm64" = "latest"          # 2026.10.9: apt keys may carry an :arch qualifier
"dnf:@development-tools" = "latest" # 2026.10.9: dnf groups
"snap:node" = { channel = "22/stable", classic = true }   # 2026.10.10: snap (pins skipped — declare a
                                    # channel; --update has no effect; uses sudo)
"apk:curl" = "*"                    # Alpine apk (version: "@2.45.2-r0" form)
"brew:postgresql@17" = "latest"     # resolves formula aliases such as `openssl`
"brew-cask:firefox" = "latest"      # app-bundle casks, no local Homebrew required
"brew-cask:1password" = { version = "latest", appdir = "/Applications" }   # 2026.10.0 per-cask appdir
                                    # (overrides MISE_BREW_CASK_OPT_APPDIR; install/upgrade only — never
                                    #  moves existing apps; first install won't replace an app unless adopt = true)
"flatpak:org.gimp.GIMP" = "latest"  # system-scoped
"flatpak-user:org.gimp.GIMP" = "latest"   # 2026.8.3+ per-user scope; both may coexist
"mas:497799835" = "latest"          # Mac App Store apps by ADAM ID
"aur:yay-bin" = "latest"            # 2026.9.2+ Arch User Repository via yay (preferred) or paru
"winget:Microsoft.PowerShell" = "latest"  # 2026.9.3+ Windows, exact package ID
"nix:ripgrep" = "latest"            # 2026.9.4+ Linux/macOS, applied through your Nix profile
"scoop:extras/vscode" = "latest"    # 2026.9.12+ Windows, adds the bucket if missing
"zypper:gcc" = "latest"             # 2026.9.12+ openSUSE / SLE

# 2026.9.11+ macOS .app from a vendor/internal download.
# version, url, sha256, artifact are ALL required — "latest" is rejected.
[bootstrap.packages."macos-app:example"]
version = "1.2.3"
url = "https://example.com/Example-{{version}}.dmg"
sha256 = "…"
artifact = "Example.app"

# Table form with version + [tools]-style os selectors (2026.8.4+).
# Platform-incompatible packages surface as unavailable instead of aborting the run.
"brew-cask:font-hack-nerd-font" = { version = "latest", os = ["macos", "linux"] }

# 2026.9.4+ `env` selector gates an entry on the active mise environment
"apt:postgresql-client" = { version = "latest", env = ["ci", "staging"] }
"pacman:libreoffice-fresh" = { state = "absent" }   # declare removal — ONLY pacman, scoop, snap,
                                    # zypper (and dnf on rpm-ostree with ostree = "layer") remove;
                                    # other managers just report drift
# 2026.10.20: Chocolatey (Windows, choco.exe on PATH; skipped elsewhere). apply/upgrade need an ELEVATED
# terminal (mise doesn't bypass UAC). Pins must match Chocolatey's version string exactly; reboot exit
# codes 1641/3010 count as success. No state = "absent", import, or prune — deleting an entry leaves it.
"choco:git" = "latest"
"choco:visualstudio2022buildtools" = "117.12.0"

# 2026.10.9: third-party apt sources (deb822 file in /etc/apt/sources.list.d/mise-<name>.sources, key
# inline; apt ≥ 2.4). `apt-get update` runs once when sources change; sources are never removed.
# A declared source REPLACES an already-installed package from elsewhere (pin a version or use an
# :arch key to opt a package out). PPAs only on Ubuntu / distros setting UBUNTU_CODENAME.
[bootstrap.apt.sources]
git-core = { ppa = "git-core/ppa" }
[bootstrap.apt.sources.docker]
uris = "https://download.docker.com/linux/ubuntu"
components = "stable"                 # default "main"; suites default = codename; also architectures
key = "https://download.docker.com/linux/ubuntu/gpg"
fingerprint = "9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88"

# 2026.10.10: Fedora COPR repos (writes what `dnf copr enable` would; never disables one)
[bootstrap.dnf.repos]
ghostty = { copr = "scottames/ghostty" }
caddy = { copr = "@caddy/caddy" }

[bootstrap.brew]
adopt = true                        # adopt existing app bundles globally (per-cask: adopt = true)
homebrew_metadata = true            # 2026.10.20, default false, Apple Silicon only: write Homebrew's cask
                                    # metadata after install so Homebrew owns the cask from then on (mise's
                                    # apply/upgrade/prune skip it; status --json reports managed_by, 2026.10.25).
                                    # Not for third-party-tap casks or a custom appdir.
taps = { "homebrew/cask-fonts" = "https://github.com/homebrew/homebrew-cask-fonts" }

# Secret inputs — stable logical names; fnox owns providers/auth
[bootstrap.secrets]
service_token = "EXAMPLE_SERVICE_TOKEN"

# Privileged files and directories
[bootstrap.files."/etc/example.conf"]
content = 'token={{ secret(name="service_token") }}'
template = true
owner = "root"
group = "root"
mode = "0644"
notify = ["example"]
phase = "pre-packages"   # 2026.9.5+ — converge this file BEFORE packages install
                         # (also valid on [bootstrap.directories])
os = "linux"             # 2026.10.5: string or list; entries for other platforms are skipped

# 2026.9.13+: targets may start with ~/ (a file you own in $HOME needs no sudo)
[bootstrap.files."~/.config/app/conf.ini"]
source = "files/app.ini.tmpl"
template = true
remove_empty = true      # an empty/whitespace render REMOVES the target (only with template = true)

# Permissions-only: omit source/content, declare mode/owner/group — manages just those, in place
[bootstrap.files."/etc/ssl/private/site.key"]
mode = "0600"
owner = "root"

[bootstrap.files."~/.oldrc"]
state = "absent"

# Linux accounts — converge BEFORE the files that reference them; UID/GID collisions fail closed
[bootstrap.users.deploy]
groups = ["docker", "www-data"]
state = "present"                   # or "absent"

[bootstrap.groups.deploy]
state = "present"

# Services — running/stopped, enabled/disabled, masked
[bootstrap.services.nginx]
state = "running"
enabled = true
# full field set: builtin, command, description, enabled, environment, masked,
#                 on_change, requires_tools, restart, scope, state, working_directory

# 2026.10.20: an EXISTING systemd user unit — scope = "user" with neither command nor builtin, managed via
# `systemctl --user` (no root; .service implied, .socket/.timer work). Only state/enabled/masked allowed.
# A missing unit or no user manager shows `unknown` and apply fails. Linux only: on macOS/Windows such an
# entry makes a full `mise bootstrap` FAIL — keep it in a Linux-only config.
[bootstrap.services.gnome-keyring-daemon]
scope = "user"
state = "running"
enabled = true

# `builtin` selects a mise-provided service: "history-watch" (dotfiles history watcher)
# or "tool-update" (2026.10.4: hourly global tool auto-updates)
[bootstrap.services.mise-history]
builtin = "history-watch"
[bootstrap.services.mise-tool-update]
builtin = "tool-update"

# Docker Compose projects (Compose v2 only) — convergence compares live container
# runtime and health against the rendered Compose model.
# 2026.10.3+: text fields are Tera-rendered in the declaring config's context
# ({{ config_root }}, {{ env.X }}); enum fields must be literal; exec() is unavailable.
[bootstrap.compose.observability]
project_dir = "{{ config_root }}/compose"   # REQUIRED, absolute after rendering (there is no `path` key)
files = ["docker/observability.yml"]
env_files = ["{{ config_root }}/compose/.env"]
state = "running"                   # running | stopped | absent
pull = "missing"                    # always | missing | never
wait = true
# also: project_name, profiles, services, oneshot, build (auto|always|never), recreate,
#       wait_timeout, timeout, remove_orphans (true), renew_anonymous_volumes, down_volumes,
#       down_images (local|all), sudo, command, engine_command, depends_on

# Linux firewall — nftables / firewalld / UFW
[bootstrap.linux.firewall]
backend = "auto"
state = "enabled"
default_incoming = "deny"
default_outgoing = "allow"
# allow_lockout = true              # required to set default-deny with no covering allow rule
# exclusive = true                  # remove undeclared rules (default: preserve them)

[[bootstrap.linux.firewall.rules]]
name = "https"
port = 443
protocol = "tcp"
action = "allow"

# Declarative git checkouts (keys may be relative paths, resolved against the project root).
# Keys (2026.10.9), url and ref (2026.10.13) accept {{ vars.* }} / {{ env.NAME }} templates
# ([env]-table values are not available).
[bootstrap.repos]
"~/src/dotfiles" = { url = "git@github.com:jdx/dotfiles.git", ref = "main" }
"{{ vars.repos_dir }}/large" = { url = "https://github.com/org/large.git", filter = "blob:none", depth = 1 }
#   2026.10.13: depth / filter / submodules apply to the FIRST clone only

# Shell activation snippets written into rc files (marker-delimited).
# Modes: "activate" | "shims" | "auto" (2026.10.5: activate in interactive fish, shims otherwise).
# Shortcut keys zsh/bash (never write ~/.zshenv), zshenv; table form { enabled = true, mode = "shims" }.
[bootstrap.mise_shell_activate]
zprofile = "shims"
zshrc = "activate"
fish = "auto"

# macOS — high-level convenience blocks…
[bootstrap.macos.dock]
autohide = true
orientation = "left"
tilesize = 48
apps = ["/Applications/Ghostty.app", "/Applications/Firefox.app"]   # 2026.9.6+

[bootstrap.macos.finder]
show_pathbar = true

[bootstrap.macos.keyboard]
key_repeat = 2
initial_key_repeat = 15

[bootstrap.macos.trackpad]
tap_to_click = true

# 2026.10.9: macOS Login Items (applied by `mise bootstrap macos defaults apply`)
[bootstrap.macos.login_items]
apps = ["/Applications/Ghostty.app", "~/Applications/X.app"]
exclusive = false                 # true = remove undeclared login items (default: additive)

# …or raw user defaults by domain
[bootstrap.macos.defaults]
"com.apple.finder" = { AppleShowAllFiles = true }

# 2026.9.5+ superset of [bootstrap.macos.defaults] — adds host/path targeting
[[bootstrap.macos.defaults_entries]]
domain = "com.apple.finder"
key = "AppleShowAllFiles"
value = true
# host = "currentHost"
# path = "~/Library/Preferences/com.example.plist"

# Services (launchd on macOS, systemd on Linux)
[bootstrap.macos.launchd.agents.my-sync]
program = "~/.local/bin/my-sync"
args = ["--watch"]
run_at_load = true
start_calendar_interval = { hour = 3, minute = 30 }
process_type = "Background"   # 2026.9.12+ → launchd ProcessType:
                              # Background | Standard | Adaptive | Interactive
                              # misspellings are REJECTED at config time
nice = 10                     # -20..20
keep_alive_on_failure = true  # KeepAlive = { SuccessfulExit = false }; not with keep_alive

# 2026.10.20: LaunchEvents, keyed by stream then an event name you choose (launchd's property case;
# quote stream names with dots; no TOML datetimes; strings templated, ~ not expanded). The program must
# consume the stream with xpc_set_event_stream_handler — this is not a general notification hook.
[bootstrap.macos.launchd.agents.usb-handler]
program = "~/.local/bin/device-handler"
[bootstrap.macos.launchd.agents.usb-handler.launch_events."com.apple.iokit.matching".apple-usb]
IOProviderClass = "IOUSBHostDevice"
idVendor = 1452
IOMatchLaunchStream = true

[bootstrap.linux.systemd.units.my-sync]
description = "sync files"
exec_start = "~/.local/bin/my-sync --watch"
exec_start_pre = ["-~/bin/check"]    # 2026.9.13: also exec_start_post, exec_stop_post (service-only lists;
                                     # ~ expansion keeps systemd prefixes like "-")
part_of = ["graphical-session.target"]   # 2026.9.13: before, binds_to, part_of, conflicts ([Unit] lists)
restart = "on-failure"
type = "simple"
remain_after_exit = false
private_tmp = true
# start = true (default); state = "absent" stops, disables, and deletes both
# dev.mise.<name>.service and dev.mise.<name>.timer (2026.10.5)

# systemd user TIMERS (2026.7.7+) — rendered as .timer units
[bootstrap.linux.systemd.units.dotfiles-maintain-timer]
unit = "dotfiles-maintain"   # bare key → dev.mise.dotfiles-maintain.service
on_calendar = "daily"
persistent = true

# Converge the login shell (runs chsh, updates /etc/shells when needed)
[bootstrap.user]
login_shell = "fish"   # 2026.10.17: a bare NAME (resolved after packages install: PATH without shims,
                       # then /opt/homebrew/bin, /usr/local/bin, linuxbrew, /usr/bin, /bin) or an
                       # absolute path like "/bin/zsh"; relative paths are ignored with a warning.
                       # Rendered as a template (no exec()). Under sudo, changes SUDO_USER's shell.

# Lifecycle hooks — run may be a string or an array; run_windows too (2026.10.18); all honor --dry-run
# Phases: pre-packages, post-packages, pre-repos, post-repos, pre-dotfiles, post-dotfiles,
#         pre-defaults, post-defaults, pre-user, post-user, pre-tools, post-tools, final
#         (underscore spellings like pre_packages also accepted). Hooks from all config files are
#         combined, global first. Skipped by --no-hooks / MISE_NO_HOOKS / safe mode. Only
#         `mise bootstrap` (and `mise dot apply` for the *-dotfiles hooks) runs them.
[bootstrap.hooks.pre-packages]
run = "softwareupdate --install-rosetta --agree-to-license"
[bootstrap.hooks.post-tools]
run = [
  "mise exec -- corepack enable",
  "mise exec -- rustup component add rustfmt clippy",
]
[bootstrap.hooks.post-defaults]
description = "Restart the Dock"   # 2026.10.22: printed instead of the command
run = "killall Dock || true"

[tools]
node = "lts"                        # Node.js 26 since 2026.10.22
python = "3.12"

# Custom task — the last app-level step, after tools install
[tasks.bootstrap]
run = "gh auth status || gh auth login"
```

> **macOS sandboxed apps (2026.9.15):** if `~/Library/Containers/<domain>` exists, defaults are read/written in the container plist (and its `ByHost`). Launch the app once first; the terminal may need Full Disk Access.

> **Shared setup repos:** `mise bootstrap --adopt` runs the shared `bootstrap` task after restoring files, but later `mise dot pull` / the history watcher restore files **without** running setup — run `mise bootstrap` again (`mise dot status` reminds you until a complete bootstrap has run).

Declarative steps converge idempotently; the `bootstrap` **task runs every invocation** (write it to be idempotent). Hooks stop bootstrap on failure and run in the current process environment. Unpinned `[bootstrap.repos]` entries are never pulled by a plain apply — use `mise bootstrap repos update` for the explicit fetch + fast-forward. Switching a systemd unit name between service and timer stops/disables/removes the stale sibling.

### Remote Bootstrap over SSH (`[bootstrap.remote]`)

Added **2026.8.2**, extended through 2026.8.11. mise archives and stages your project on each target, provisions a compatible mise binary, runs bootstrap with forwarded flags, then cleans up staging.

```toml
[bootstrap.remote]
source = "."                        # local directory archived and sent to each target
mise_env = ["production"]           # config environments to load remotely (2026.8.10+)
install_mise = true                 # persist mise on the host (default ~/.local/bin/mise)
copy_links = false                  # dereference every symlink in the archive
copy_link = ["config/secrets"]      # or dereference selected source-relative links
exclude = ["node_modules", ".git"]

[bootstrap.remote.hosts.web1]
host = "web1.internal"
user = "deploy"
port = 22
identity_file = "~/.ssh/id_ed25519"
tags = ["web", "prod"]
ssh_options = ["StrictHostKeyChecking=yes"]
# per-host overrides: source, mise_env, copy_links, copy_link, exclude,
#                     install_mise, mise_bin, remote_mise, bootstrap_command
```

```bash
mise bootstrap remote web1 db1          # named inventory hosts
mise bootstrap remote --all
mise bootstrap remote --tag prod        # select by tag (repeatable, matches any)
mise bootstrap remote --host deploy@10.0.0.5   # ad-hoc target
mise bootstrap remote -n                # dry-run
mise bootstrap remote --remote-env ci,dotfiles
mise bootstrap remote --install-mise=/usr/local/bin/mise
mise bootstrap remote --only packages,dotfiles --fail-fast
mise bootstrap remote --prompt-secrets --keep-staging
```

**Binary provisioning is fail-closed.** mise detects each target's OS, architecture, and Linux libc (glibc vs musl); when the local binary is incompatible it downloads the matching raw executable for the *same release* from GitHub with **minisign-verified checksums**. Custom or debug builds refuse to guess and require an explicit `mise_bin`, `remote_mise`, or `bootstrap_command`.

`--only` / `--skip` accept the same part names as local bootstrap. Other flags: `--connect-timeout`, `--force-dotfiles`, `--update`, `-y/--yes`, `-i/--identity-file`, `--port`, `--ssh-option`, `--bootstrap-command`, `--adopt`, `--source`, `--exclude`, `--copy-link(s)`, `--remote-mise`, `--mise-bin`, `--no-install-mise`, and the `--github-relay-*` flags (see [`mise ssh`](dev-tools.md#cli-commands-for-tools)).

### Declarative Dotfiles (`[dotfiles]`)

Manage dotfiles declaratively; applied during `mise bootstrap` (step 10) or standalone via **`mise dot apply`**. **Stable since v2026.7.4.** Entries are keyed by target path. Docs are split into `/dotfiles/{managed,edits,encryption,groups,history,sync,reference}.html`.

> **`mise dotfiles` / `mise dot` is THE command.** It was un-deprecated in 2026.9.8, and since **2026.10.5** the old `mise bootstrap dotfiles …` spelling is **hidden** (still works as an alias; its doc pages were deleted). Subcommands: `apply`, `status`, `add`, `diff`, `edit`, `unapply`, `conflicts`, `recover`, `capture`, plus the history system (`save`, `history` incl. **`history purge`**, `rollback`, `undo`, `track`, `untrack`, `sync`, `origin`, `watch`, `paths`, `pull`, `exclude`, `include`) and **`notify`** (2026.10.4 — send a test desktop notification for `history.notify`). Global flag `mise dot --local <subcommand>` targets local-only history. Undoable `mise dot` operations no longer ask for confirmation (2026.10.5).

```toml
[settings]
dotfiles.root = "~/.dotfiles"      # default source root; templates allowed (2026.10.20): "{{ mise_config_dir }}/home"
dotfiles.default_mode = "symlink"  # symlink|symlink-each|copy|template

[dotfiles]
"~/.zshrc" = {}                                   # source mirrors target under dotfiles.root
"~/.gitconfig" = "dotfiles/gitconfig"             # string shorthand = source path
"~/.config/alacritty.toml" = { mode = "copy" }
"~/.ssh/config" = { source = "dotfiles/ssh_config.tmpl", mode = "template" }
"~/.local/bin" = { source = "dotfiles/bin", mode = "symlink-each", exclude = ["*.bak"] }
"~/.config/*.toml" = "dotfiles/config/*.toml"     # glob: * ** ? [ab] (target must match)
"~/.config/small.conf" = { content = "key = value\n" }   # 2026.8.6+ inline body;
                                                  # cannot combine with source/mode/exclude
# Block / line edits to files mise does NOT own (key = target/edit-id)
"~/.zshrc/activate" = { block = 'eval "$(mise activate zsh)"' }
"/etc/hosts/dev" = { line = "127.0.0.1 dev.local", position = "prepend" }  # append (default) | prepend
"~/.gitconfig/identity" = { source = "snippets/git-identity.tmpl", template = "tera" }

# Encrypt tracked contents in Git; the LIVE file stays plaintext (TRACK entries only —
# encrypt has no effect on symlink/copy/template entries)
"~/.config/app/token" = { mode = "track", encrypt = true }

# 2026.10.4/10.5: MERGE edit — own only the keys present in the source within a JSON/JSONC/TOML/
# YAML/INI target (JSONC 2026.10.22 and INI 2026.10.23 are edited in place, keeping comments; key order
# preserved for TOML/YAML); "missing" only adds absent keys; unapply leaves merged keys in place;
# source may be omitted for merge entries. `format` (json|jsonc|toml|yaml|ini) is needed when the
# extension doesn't say: a .json target without it is STRICT JSON (comments rejected).
# `permissions` on a merge applies only when the merge creates the file.
"~/.codex/config.toml/shared" = { source = "codex/shared.toml", merge = true }
"~/.config/tool/settings.json/defaults" = { source = "tool/defaults.json", merge = "missing" }
"~/.config/Code/User/settings.json/shared" = { source = "vscode/shared.json", merge = true, format = "jsonc" }
"~/.config/pip/pip.conf/shared" = { source = "pip/shared.conf", merge = true, format = "ini" }

# 2026.10.22: create_if_missing — write a copy/template/content entry only when nothing exists there
"~/.config/mise/conf.d/private.local.toml" = { source = "templates/private.toml", mode = "copy", permissions = "0600", create_if_missing = true }
# 2026.10.23: fnox mode — contents come from fnox (>= 1.44.0; `fnox set --file ~/.npmrc` first)
"~/.npmrc" = { mode = "fnox" }

# 2026.10.5: local-only history (stored in $MISE_STATE_DIR/history-local, never pushed to origin;
# rejects encrypt and variants) — CLI: mise dot track --local <path>
"~/.config/app/state.json" = { mode = "track-local" }
# 2026.10.5: a per-machine stream (still pushed); stream name via [history] machine = "desk"
"~/.config/monitors.xml" = { mode = "track", variants = [{ machine = true }] }   # mise dot track --machine
# also: enabled = false (skip an entry), autosave = false

# Select managed files from a directory via its git manifest (copy / symlink-each only)
"~/.config/nvim" = { source = "dotfiles/nvim", mode = "symlink-each", manifest = "git" }

# 2026.9.13+: delete a file on every machine that applied the config (file or symlink only, no globs)
"~/.oldrc" = { mode = "absent" }
# 2026.9.13+: permissions on copies/templates/inline content — or ALONE, managing only an existing target's mode
"~/.netrc" = { source = "netrc", mode = "copy", permissions = "0600" }
"~/.ssh" = { permissions = "0700" }
# 2026.9.13+: remove the target when the template renders empty (only what mise last wrote)
"~/.config/work.env" = { source = "work.env.tmpl", mode = "template", remove_empty = true }
# 2026.9.14+: GNU Stow style — dot-<name> sources deploy as .<name>; relative symlinks
"~" = { source = "home", mode = "symlink-each", dot_prefix = true, relative = true, exclude = ["README.md"] }
# Track mode: select what is captured (2026.9.13) — explicit exclude always wins
"~/.codex" = { mode = "track", include = ["config.toml", "rules/**"], exclude = ["rules/tmp/**"] }

# Per-OS / per-profile variants
[[dotfiles."~/.gitconfig".variants]]
os = ["macos"]
target = "~/.gitconfig-mac"
[[dotfiles."~/.gitconfig".variants]]
profile = "work"
default = true
```

**Entry fields:** `source` · `content` · `mode` (`symlink` \| `symlink-each` \| `copy` \| `template` \| `track` \| **`track-local`** \| **`absent`**) · `exclude` · `manifest` (`"git"`) · `block` · `line` · `position` (`append` \| `prepend`) · `template` (`"tera"`) · **`merge`** (`true` \| `"missing"`) · `comment` · `encrypt` (track only) · `variants` (each `{ os?, profile?, default?, target?, machine? }`; `target` is a Tera template since 2026.10.18) · **`permissions`** (octal string `"0600"`) · **`remove_empty`** (template mode) · **`relative`** (symlink modes) · **`dot_prefix`** (directory source + `symlink-each`/`copy`) · **`include`** / **`allow_plaintext`** (track only) · `enabled` · `autosave` · **`depends`** (2026.11.0, below). A template entry with no source falls back to `<path>.tera` (2026.10.12).

| New field (2026.9.13–9.16) | Rules |
|----------------------------|-------|
| `mode = "absent"` | Deletes a regular file or symlink (never follows the link; never removes a dir/socket/FIFO — errors even with `--force`). No `source`/`content`/edit keys, no globs. Works with destination `variants` (e.g. remove only on macOS). Status `absent` once gone, `would remove` while present. `mise dot unapply` skips it; `mise oci build` adds a whiteout. **This is the way to clean up a file you stopped managing** — merely deleting the entry leaves it in place. |
| `mode = "fnox"` (2026.10.23) | Whole-file contents stored in fnox (needs fnox ≥ 1.44.0 on PATH). No `source`/`content`/`exclude`/`manifest`/`encrypt`/`remove_empty`/`create_if_missing`/`merge`/edit keys and no glob targets; accepts `permissions` (default `0600`) and `variants`. `mise dot edit` runs `fnox edit --file`; `mise dot add` won't capture it; `diff` never prints content; `--dry-run` never calls fnox; `unapply` removes it only while it still matches fnox unless `--force`; `mise oci build` skips it. |
| `create_if_missing` (2026.10.22) | `copy`/`template` file sources and inline `content`. Writes only when nothing exists at the target; an existing file (or symlink) is left alone and reports `applied`; a deleted target is re-created; a directory at the target is a conflict. `unapply` leaves it unless `--force`. |
| `permissions` | Works with `copy`, `template` (file source), inline `content`, `fnox`, and `merge` edits (only when the merge creates the file, 2026.10.22). **On its own** it manages only an existing target's permissions (missing target → warning, status `applied` with reason `target absent; permissions not applied`). Not with `symlink`/`symlink-each`/`track`/directory sources; ignored on Windows. |
| `remove_empty` | Template mode only. An empty/whitespace render removes the target if it still holds what mise last wrote (else conflict; `--force` overrides), plus now-empty parent dirs mise created inside `$HOME`. |
| `relative` | Relative symlink targets; requires `symlink`/`symlink-each`; overrides the `dotfiles.relative_symlinks` setting (default `false`). Ignored on Windows (junctions). |
| `dot_prefix` | Stow `--dotfiles`: source components `dot-<name>` deploy as `.<name>`. `exclude`/`manifest` match **source** names. A `dot-bashrc` + `.bashrc` collision fails. `mise dot add` refuses to capture into a `dot_prefix` entry (groups accept). |
| `include` (track) | Only matching paths are captured; `include = []` selects nothing. Selects credential-named files too (plaintext unless `encrypt = true`). New checkpoints use history schema v2, so **upgrade every machine** sharing the history first. |
| `allow_plaintext` (track) | Written by `mise dot track --allow-plaintext` — lets a directly tracked credential-named file be stored in plaintext. `--yes` does **not** imply it. |

**Ordering with `depends` (2026.11.0).** `mise dot apply` normally plans every entry before writing any, so a template or `fnox` entry can't read a file the same apply creates. `depends` lists dotfile **groups** (by name) and **whole-file entries** (by target path) to write first; mise applies in rounds.
```toml
[dotfile_groups.secrets]
root = "secrets"
mode = "copy"
target = "~/.config/fnox"

[dotfiles]
"~/.workrc" = { mode = "fnox", depends = ["secrets"] }
"~/.npmrc" = { mode = "fnox", depends = ["~/.ssh/encryption_ed25519"] }
"~/.ssh/encryption_ed25519" = { mode = "template", source = "ssh/encryption_ed25519.tera" }
```
- A group's `depends` also applies to its entries. A dependency the apply leaves out (e.g. not named in `mise dot apply <target>`) counts as already applied.
- An unknown name or a cycle is an error. If a round fails, earlier rounds stay written and mise lists what wasn't applied.
- `--dry-run` lists dependent entries without rendering them; the confirmation prompt appears once per round. Edits and `track` entries take no `depends`.

**Pattern matching (2026.9.15):** a leading `/` anchors a pattern to the entry root like `.gitignore` (`"/cache"` = top level only); `**` crosses directories; in `include` patterns `*` **never** crosses `/`. In `exclude` patterns containing `/`, `*` still crosses `/` but **warns** (deprecated 2026.9.13, removal **2027.9.13**) — write `**` instead.

- `encrypt = true` (track entries only) stores encrypted contents in Git while the live file stays plaintext. Shared recipients come from `[history].encryption.recipients`; re-encrypt existing history with `mise dot save --re-encrypt` (2026.10.6).
- `variants` are history **streams** in track mode, or optional **destinations** in deployment modes. A variant `target` needs an explicit `source` (or a safe relative entry key when every variant has a target) and is **not supported in track mode**.
- Templates (`mode = "template"`) can reference `[bootstrap.secrets]` via `{{ secret(name="…") }}` (2026.9.7+), and so can `merge = "missing"` + `template = "tera"` edits (2026.10.24; they prompt only for keys missing from the target, `status`/`diff` never prompt, and `mise bootstrap` doesn't preflight them, so an unavailable secret stops the run at that entry; enforced `merge = true` can't use `secret()`). Commands that render templates (`add`, `apply`, `diff`, `edit`, `status`, `unapply`) accept `--prompt-secrets`; without a value, rendering **fails closed**. Textual diffs redact resolved secret values.

> 🔒 `mise oci build` renders dotfile templates with a **restricted engine**: `secret()` is rejected and the `env` context, `get_env()`, `exec()`, and `read_file()` are unavailable, so ambient credentials cannot be baked into a publishable image layer.

**Modes:** `symlink` (default; one link for a file or whole directory) · `symlink-each` (directory source → per-file links, so the target dir can also hold unmanaged files; supports `exclude` globs, prunes stale mise-managed links, and records exact source→target pairs in a manifest under `$MISE_STATE_DIR/dotfiles` rather than recursively walking shared targets like `~`) · `copy` (a real file/dir; additive for directories and **never pruned**) · `template` (render the source through mise's Tera engine with `env`, `vars`, `exec()`; permissions mirror the source and are repaired on drift).

**Source resolution:** omitting `source` mirrors the home-relative target path under `dotfiles.root` (`~/.zshrc` → `~/.dotfiles/.zshrc`). Relative explicit sources resolve against the declaring config file's directory. Targets outside `$HOME` require an explicit `source`.

**Block edits** wrap content in marker comments (`# >>> mise:id >>>` … `# <<< mise:id <<<`); the comment style is inferred per file type (`#`, `--`, `//`, `;`, `"`) and can be overridden with `comment`. Strict JSON/XML cannot use blocks. **Line edits** append a single line if absent and never modify other content. Edit IDs allow letters, digits, `_`, `-`, `.`. A table with `source` + `template = "tera"` is unambiguously an edit; a table with only `source` is a whole-file entry.

```bash
mise dot status [--missing] [--json] [-v]   # default view lists only entries needing attention, grouped
                                            # into sections (2026.10.9/10.10) — scripts: use -v or --json
                                            # (alias: ls); JSON adds `reason` on differs + permission-only
mise dot apply [--dry-run] [--verbose] [--yes] [--force]
mise dot apply --prune                      # 2026.10.3: also remove files from deselected/deleted groups
mise dot add ~/.zshrc [--no-apply]          # capture a live file (applies by default); refuses a
                                            # mode = "template" target unless --force (2026.10.21)
mise dot add ~/.config/starship.toml --group home   # 2026.10.3: capture into a dotfile group
mise dot diff                               # changes needed to apply
mise dot edit [--apply] [--group NAME] ~/.zshrc
mise dot unapply [--group NAME]             # remove managed links/copies/templates/blocks (one group)
mise dot conflicts [PATH…] [--difftool]     # 2026.9.7+ inspect local vs remote before resolving
mise dot recover                            # recover an interrupted dotfile operation
mise dot capture                            # capture live changes back into sources
```

Status values: `applied`, `missing`, `differs`, `absent`, `would remove`, `source missing`, `tracked`, `not on this machine`, `ambiguous variants`, `orphaned`, `orphaned (changed since applied)`.

`mise dot apply`, the bootstrap dotfiles phase, and `mise dot pull` also run `[history.reload]` commands. Only `mise dot apply` and the bootstrap dotfiles step run the `pre-dotfiles`/`post-dotfiles` hooks (`dot add` and `dot edit --apply` don't). `mise bootstrap dotfiles …` is a hidden alias of `mise dot …`.

`conflicts` is read-only — a unified diff including file-mode changes by default, `--difftool` opens the configured Git `diff.tool` (falling back to `merge.tool`), `--tool <name>` picks one. Encrypted contents decrypt only into private temporary files, and inspection never modifies either side or marks the conflict resolved. Resolve with `--take-remote` or `--keep-local`.

Dotfiles are **manual-only** — never applied implicitly by `mise install` or `mise bootstrap packages`. A **real file at the target always needs `--force`** to become a symlink, even when its content matches. `--dry-run` promises to execute nothing, so it **skips template rendering** and lists those entries as `(if changed)`. On Windows, `symlink` copies unless Windows allows the link, and `symlink-each` copies unless `dotfiles.windows_symlinks = true` (2026.10.15); directories use junctions. Safe mode ignores project `[dotfiles]`/`[dotfile_groups]`. Removing a config entry leaves files in place — replace it with `mode = "absent"` (or `[bootstrap.files."~/x"] state = "absent"`) to delete it on every machine, or run `unapply` locally.

### Dotfile Groups (`[dotfile_groups]`)

Added **2026.10.3**. Stow-package-style **named directory trees** under `dotfiles.root`, each deployed as a unit and selectable per machine:

```toml
[dotfile_groups.zsh]
root = "zsh"                 # REQUIRED; relative = under dotfiles.root (~/.dotfiles), NOT the config dir

[dotfile_groups.home]
root = "home"
target = "~"                 # default "~"; absolute or ~/
mode = "symlink-each"        # default; symlink-each | copy | symlink (whole tree)
dot_prefix = true            # default false
exclude = ["README.md"]
# manifest = "git"           # only files in Git's index
# relative = true            # overrides dotfiles.relative_symlinks

[dotfile_groups.home.entries]               # whole-file entries cut out of the walk
"~/.config/kitty" = { mode = "symlink" }    # link a directory as a whole
"~/.ssh/config" = { mode = "copy", permissions = "0600" }
"~/.gitconfig" = { source = "git/config.tmpl", mode = "template" }   # relative source starts at the group root
"~/.kitty-old.conf" = { mode = "absent" }

# Per machine, e.g. ~/.config/mise/config.local.toml
[bootstrap]
dotfile_groups = ["home", "zsh"]   # unset = ALL declared groups apply; [dotfiles] entries always apply
```

**Groups without a table (2026.10.21):** a name in `[bootstrap] dotfile_groups` with no `[dotfile_groups.<name>]` table but a matching directory under `dotfiles.root` becomes a default group (that directory, target `~`, `symlink-each`). These exist **only while a list selects them**; a declared table always wins.

**Edits in groups (2026.10.23):** `[dotfile_groups.<n>.entries]` can hold block, line, and merge edits keyed `<path>/<id>`; they apply only while the group is selected:

```toml
[dotfile_groups.common]
root = "common"

[dotfile_groups.common.entries]
"~/.config/mise/conf.d/private.local.toml/defaults" = { source = "../templates/private.toml", merge = "missing", permissions = "0600" }
```

The edited file must be inside the group's target (no `..`); a relative `source` starts at the group root; a whole-file-only key such as `mode` on an edit makes mise **ignore the whole group** with a warning. Deselecting the group leaves what the edit wrote — run `mise dot unapply <path>` while the group is still selected to remove a block or line.

- Group names match `^[A-Za-z0-9_.-]+$`; a more-local config with the same group name **replaces** the whole group table.
- A group's `depends` (2026.11.0) names groups and entry targets to write first — see [Ordering with `depends`](#declarative-dotfiles-dotfiles).
- Two selected groups may not deploy the same file, nor may one link a directory whole while another places files in it — the conflict is reported before anything is written.
- Entries without `source` find it under the group root at the same relative path; entries without `mode` deploy like the group; walking entries inherit `dot_prefix`, `manifest`, and slash-less `exclude`s. Every entry must lie inside the group's `target`.
- **Deselecting or deleting a group leaves its files** — mise records deployments under `$MISE_STATE_DIR/dotfiles/groups` and `mise dot status` lists them as **`orphaned`**. Clean up with `mise dot apply --prune` (every group; asks unless `--yes`) or `mise dot unapply --group <name>`. Both only remove links still pointing at the source / copies still holding what mise wrote (`--force` for changed copies), and never remove through linked dirs or inside `dotfiles.root`.
- `mise dot add <file>` captures into the deepest containing group (choose with `--group` for a new file); `manifest = "git"` groups refuse `add`. `mise bootstrap --prune-dotfiles` (2026.10.15) also removes orphaned files.
- **Switching from GNU Stow (2026.10.21):** a directory Stow folded into one link is replaced by a real directory of per-file links without `--force`; a link into the group root that points at the wrong directory is refused even with `--force`.
- `mise oci build` does **not** include dotfile groups.

### Dotfiles History (`mise dot` / `[history]`)

mise saves versions of tracked configuration files as **ordinary Git commits** ("checkpoints") so you can inspect changes and restore an earlier version. History stays **local** until you connect a remote setup repository.

```bash
mise dot track ~/.zshrc                       # tracks and saves the first version immediately
mise dot track ~/.config/app/state.json --no-autosave
mise dot save ~/.zshrc                        # save one file
mise dot save --description "before theme change"   # save all tracked files
mise dot untrack ~/.zshrc
mise dot paths [--noisy]                      # what history tracks, and under which policies
mise dot track --dry-run ~/.config/app        # 2026.9.13: preview files/size/what is left out
mise dot paths --preview ~/.config/app        # same preview; warns above 5,000 files or 256 MiB
mise dot track --allow-plaintext ~/.netrc     # 2026.9.16: store a credential-named file unencrypted
mise dot files ~/.config                      # 2026.11.5: each file's state — tracked, excluded,
                                              # omitted, untracked, partial, local (-r/--recursive,
                                              # --tracked, --untracked, -J/--json)
```

`track`/`untrack`/`add`/`edit` accept relative paths (home paths are stored as `~/…`, 2026.11.5). `mise dot sync` merges exclusions added on two machines (2026.11.4); removals and `!glob` changes on both sides still conflict.

**Credential filtering is by filename, not contents:** `.netrc`, `*.age`, `*.key`, `*.pem`, `*.gpg`, `*.kdbx`, `id_*` (including `id_ed25519.pub`), `*token*`, `*secret*`, `credentials*`, `oauth*`, and under the mise config dir `github_tokens.toml`, `hosts.yml`, `age.txt`. `*.local.toml` is **always** omitted, even with encryption. `mise dot paths` lists omissions with reasons.

**Nested repositories (2026.9.13):** a directory containing `.git` inside a tracked directory is skipped and reported — track its root explicitly to save its working files (`.git` is always excluded).

An ordinary `save` with no changes creates no commit; supplying `--description`, `--label`, or `--task` creates a checkpoint anyway. `save` **fails** if it can't save anything (Git missing, history disabled, path untracked) — use `--best-effort` in scripts to warn and continue.

**The watcher** is a background service saving changes made by any editor or command — a systemd user service on Linux, a LaunchAgent on macOS, a Scheduled Task on Windows:

```toml
[bootstrap.services.mise-history]
builtin = "history-watch"
```

```bash
mise bootstrap services apply
mise dot status
mise dot watch --once
```

**Comparing and restoring:**

```bash
mise dot history [--path ~/.zshrc]            # checkpoints, newest first
mise dot history show 12 [--files] [--json]
mise dot history diff 12 --patch --path ~/.zshrc
mise dot history diff 11 12 --patch           # between two checkpoints
mise dot history diff --path ~/.zshrc         # current file vs latest saved (add --exit-code)
mise dot rollback ~/.zshrc [--dry-run]
mise dot rollback --to latest~3 --all --dry-run
mise dot undo                                 # reverse the tracked-file changes of an operation
```

| Reference | Meaning |
|-----------|---------|
| `12` | Checkpoint with local ID 12 |
| `latest` | Most recent checkpoint |
| `latest~N` | N checkpoints back (with `--path`, counts only checkpoints where that path changed) |
| `commit:<sha>` | Git commit by full hash or unambiguous prefix |

Numeric IDs are **local and may change** if mise rebuilds its index — use a Git commit hash for a stable reference. Rollback saves current contents first, so `undo` can reverse it; both create new commits, so earlier versions stay available. `--all` selects everything in the chosen checkpoint, and a checkpoint that recorded a file as **absent** makes rollback **delete** it. Preview actions are `write`, `delete`, `remove if empty`, `unchanged`, `skip`, and `conflict`.

**Erasing a path from history (2026.10.18, destructive):** `mise dot untrack <path>`, stop the watcher, then `mise dot history purge [-n] [-y] [--backup] [--keep-objects] <PATH>…` — rewrites local history; refuses if the path was ever pushed to the origin; `--backup` keeps the old history under `refs/mise/purge-backup/`. Works with `mise dot --local`.

**Sharing** connects a setup repository:

```bash
mise dot origin set <url>
mise dot sync                                  # publish, fetch, record what is pending
mise dot pull [--keep-local-all]               # restore incoming tracked paths into live files
                                               # (no PATH args since 2026.10.5)
```

`sync` **scans for secret-looking content before pushing** (2026.10.5) — bypass with `--allow-plaintext-history` or `history.allow_plaintext_history`. `mise dot untrack` on a path inside a tracked directory adds an exclude rule.

**`[history]` configuration:**

```toml
[history]
exclude = ["**/*.log", "!important.log"]       # last matching rule wins; a !glob re-includes inside an
                                               # excluded dir — but cannot override a per-entry `exclude`
git_email = "mise@{hostname}"                  # 2026.9.17: author/committer for saves, autosaves, merges
                                               # ({hostname} expands per machine; default mise@localhost)

[history.encryption]
recipients = ["age1…", "ssh-ed25519 AAAA…"]    # shared age/SSH/age-plugin recipients

[history.reload]
"~/.zshrc" = "exec zsh"                        # run once after a rollback/undo/pull writes a matching path
                                               # (trusted GLOBAL or SYSTEM config only)

[history.origin]                               # machine-local; written by `mise dot origin set`
url = "git@github.com:me/setup.git"
branch = "main"
```

**`[settings.history]`:**

| Setting | Default | Description |
|---------|---------|-------------|
| `enabled` | `true` | Record Git commits for tracked files around bootstrap operations and on explicit saves |
| `sync` | `"sync"` | What the watcher does with a connected setup repo on its own: `sync` \| `fetch-only` \| `manual` |
| `sync_interval` | `"5m"` | How soon after a save the watcher publishes, at most this often |
| `fetch_interval` | `"15m"` | How often the watcher fetches the origin branch (values below 1s use 1s) |
| `notify` | `true` | Desktop notification when conflicts pause sharing (retries stay silent) |
| `allow_plaintext_history` | `false` | Allow publishing history containing unencrypted versions of files now marked for encryption (also bypasses the pre-push secret scan) |
| `describe_command` | `""` | Command that names commits the watcher saves (an agent, say); gets one JSON object on stdin |
| `watch.debounce` | `"2s"` | Quiet period before the watcher saves a changed file |
| `watch.max_interval` | `"24h"` | Longest autosave interval a constantly-changing file is stretched to |
| `watch.reconcile` | `"10m"` | Full rescan for missed changes (`0` disables periodic reconciliation) |

Ordinary edits are saved after ~2s of quiet; constantly-changing files are stretched on their own schedule (doubling up to `max_interval`) and never delay the others. Explicit `save` is always immediate.

> 🔒 **`history.describe_command` is global-only (2026.9.7 security fix).** An implicitly trusted project could previously set it and have a later checkpoint execute the project-controlled command with **unencrypted tracked-file diffs**. It is now honored only from system/global config or `MISE_HISTORY_DESCRIBE_COMMAND`; project values are ignored with a warning.

> **Watcher self-restart (2026.9.18):** the history watcher checks every minute whether its executable was replaced; it saves pending work and exits non-zero so the service manager restarts it on the new version. Pre-safeguard watchers need one `mise bootstrap services apply` (`mise doctor`/`mise dot status` report a "stale watcher schema"). Warnings from background captures are stored and shown at the next `mise dot` command or `mise bootstrap`.

> Logs, caches, databases, and session state usually belong **outside** tracked files. For a file you want saved only on request, track it with `--no-autosave` and use explicit `mise dot save`.
