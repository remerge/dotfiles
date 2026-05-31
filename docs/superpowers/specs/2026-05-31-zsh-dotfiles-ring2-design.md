# Remerge dotfiles — Ring 2 (everyday CLI niceties) design

**Date:** 2026-05-31
**Status:** Approved (design); ready for implementation planning
**Repo:** <https://github.com/remerge/dotfiles>
**Builds on:** Ring 0 + Ring 1 (merged) — see
`docs/superpowers/specs/2026-05-28-zsh-dotfiles-skeleton-design.md`

## Goal

Extend the merged zsh skeleton with the "everyday CLI niceties" from the Ring 1
roadmap, staying a faithful **subset** of <https://github.com/hollow/dotfiles>:
a `diff` against upstream should show only deletions, the previously-trimmed
files, and a small set of clearly-flagged intentional deviations.

Ring 2 adds a handful of modern CLI tools, a few shell niceties, impersonal git
defaults, and a clean per-user git-identity mechanism. fzf, atuin, git signing,
and personal git aliases remain deferred.

## Scope (decided)

In scope:

- **Modern CLI tools:** `eza`, `bat`, `fd`, `ripgrep`, plus `duf`, `rsync`,
  `wget`, and `glow`/`glamour`.
- **Shell niceties:** pager/`less` config, `colored-man-pages`, `you-should-use`,
  `dircolors` (LS_COLORS).
- **git:** impersonal `git/config` defaults, the global `git/ignore`, the 8
  self-contained git aliases, and per-user identity via a git `[include]`.

Out of scope (still deferred): fzf/fzf-tab, atuin, git signing &
identity-in-tracked-config, the personal git aliases that depend on dropped
`git-*` subcommands, and a general zsh personal-override layer.

## Faithfulness principle (carried over)

Every kept line stays byte-identical to upstream. `zsh/.zshrc` remains a strict
line-subset (only upstream lines, re-inserted at their original relative
positions between the brew bootstrap and the starship block). Deviations are
explicit and enumerated below.

### Intentional deviations (net-new, not in upstream)

1. `brew "glow"` in the `Brewfile` — upstream's `Brewfile` does not list glow
   (its glow config exists but it isn't brew-installed there). The repo owner
   will add glow upstream, after which this returns to a subset.
2. `git/config` ends with an `[include] path = local` directive (see below).
3. `git/local.example` — a tracked template for per-user git identity.
4. `git/.gitignore` — ignores the untracked `git/local` (follows the upstream
   per-directory `.gitignore` convention, e.g. `auth0/.gitignore`).

## File inventory

### Modify

- `Brewfile` — add `bat`, `duf`, `eza`, `fd`, `glow`, `ripgrep` (alphabetical;
  `rsync`/`wget` already present from Ring 1).
- `zsh/.zshrc` — add the tool/nicety/git blocks below (strict subset).

### Create — vendored verbatim from `hollow/dotfiles@main`

- `bat/config`
- `git/ignore`
- `wgetrc`
- `glow/glow.yml`
- `glow/styles/catppuccin-mocha.json`

### Create — net-new (intentional deviations)

- `git/config` — impersonal settings + the `[include]` directive.
- `git/local.example` — commented `[user]` identity template.
- `git/.gitignore` — `/local`.

### Unchanged / not added

- `ripgrep` and `fd` are **Brewfile-only** — upstream removed their no-op
  `zi auto … for` null-plugin lines, so there is no `.zshrc` block for either.

## `zsh/.zshrc` additions

All blocks below are byte-identical to upstream and inserted preserving
upstream's relative order (between `zi auto has"dscl" for brew` and the starship
block).

**bat:**

```zsh
# bat: cat(1) clone with wings
# https://github.com/sharkdp/bat
:bat-load() {
    export BAT_CONFIG_PATH="${XDG_CONFIG_HOME}"/bat/config BAT_PAGER="less"
    export MANPAGER="sh -c 'col -bx | bat -l man'" MANROFFOPT="-c"
}

zi auto has"bat" wait for bat
```

**dircolors:**

```zsh
# dircolors: setup colors for ls and friends
# https://github.com/trapd00r/LS_COLORS
:dircolors-load() {
    zstyle ":completion:*:default" list-colors "${(s.:.)LS_COLORS}"
}

:dircolors-eval() {
    dircolors -b LS_COLORS
}

zi auto id-as"dircolors" wait for trapd00r/LS_COLORS
```

**duf:**

```zsh
# duf: better `df` alternative
# https://github.com/muesli/duf
:duf-load() {
    alias df=duf
}

zi auto has"duf" wait for duf
```

**eza:**

```zsh
# eza: a modern replacement for ‘ls’.
# https://github.com/ogham/eza
:eza-load() {
    alias l="eza --all --long --group"
    alias lR="l -R"
}

zi auto has"eza" wait for eza
```

**git (completion + the 8 self-contained aliases only):**

```zsh
# git: distributed version control system
# https://github.com/git/git
zi auto id-as"git" as"completion" blockf mv"git->_git" wait for \
    https://github.com/git/git/blob/master/contrib/completion/git-completion.zsh

alias ga="git add --all"
alias gap="git add --patch"
alias gd="git diff"
alias gf="git fetch --prune"
alias gp="git pull"
alias gpr="git pull --rebase --autostash"
alias grh="git reset HEAD"
alias gsp="git show -p"
```

Dropped (need dropped `git-*` subcommands or git config aliases): `gcl`, `gcm`,
`gcu`, `gdc`, `gdm`, `gdu`, `gl`, `s`, and the `git-each`/`git-parallel` aliases.

**glamour/glow:**

```zsh
# glamour/glow
export GLAMOUR_STYLE="${HOME}/.config/glow/styles/catppuccin-mocha.json"
export GLOW_STYLE="${GLAMOUR_STYLE}"
```

**less / pager:**

```zsh
# less: pager configuration
# https://man7.org/linux/man-pages/man1/less.1.html#OPTIONS
export PAGER="${commands[less]}" LESS="--ignore-case --LONG-PROMPT --RAW-CONTROL-CHARS --HILITE-UNREAD --chop-long-lines --tabs=4"
export LESSHISTFILE="${XDG_DATA_HOME}/less/history"
mkdir -p "$(dirname "${LESSHISTFILE}")"
```

**colored-man-pages:**

```zsh
# man: unix documentation system
# https://www.nongnu.org/man-db/
zi auto wait for OMZP::colored-man-pages
```

**rsync:**

```zsh
# rsync: fast incremental file transfer
# https://rsync.samba.org
zi auto wait for OMZP::rsync
```

**wget:**

```zsh
# wget: retrieve files using HTTP, HTTPS, FTP and FTPS
# https://www.gnu.org/software/wget/
export WGETRC="${XDG_CONFIG_HOME}/wgetrc"
alias wget="wget --hsts-file=\"${XDG_CACHE_HOME}/wget-hsts\""
```

**you-should-use:**

```zsh
# reminds you to use existing aliases for commands you just typed
# https://github.com/MichaelAquilina/zsh-you-should-use
if has tput; then
    zi auto wait for MichaelAquilina/zsh-you-should-use
    YSU_MESSAGE_POSITION="after"
fi
```

## `Brewfile` additions

Final brew list (alphabetical) gains `bat`, `duf`, `eza`, `fd`, `glow`,
`ripgrep`. All exist in upstream's `Brewfile` **except `glow`** (the single
flagged deviation, pending the upstream add).

## git identity via `[include]`

`git/config` is tracked and impersonal, ending with:

```ini
[include]
    path = local
```

A relative include `path` resolves against the including file's directory, so
this reads `~/.config/git/local`. That file is per-user and untracked, holding
only personal identity:

```ini
# ~/.config/git/local  (yours; never committed)
[user]
    name = Your Name
    email = you@remerge.io
    # signingkey = ...        # optional, later
# [commit]
#     gpgsign = true          # optional, later
```

- A **missing** include is silently ignored by git, so before identity is set,
  git shows its normal "tell me who you are" prompt on first commit — no errors,
  no fake identity.
- `git config user.email` reflects the real value once set; signing is a
  future drop-in.
- `install.sh` copies `git/local.example` → `git/local` on first install if
  absent, giving fresh users a ready-to-edit file. `zsh/.zshrc` is untouched.
- `git/.gitignore` (`/local`) keeps the per-user file out of version control.

**Impersonal `git/config` contents** (each line byte-identical to upstream,
personal/signing sections omitted, plus the `[include]`):

```ini
[advice]
	detachedHead = false

[branch]
	sort = -committerdate

[color]
	ui = true

[diff]
	renames = copies

[init]
	defaultBranch = main

[pull]
	ff = only

[push]
	followTags = true
	autoSetupRemote = true

[rerere]
	enabled = true

[include]
	path = local
```

Omitted from upstream's `git/config`: `[user]`, `[gpg]`, `[commit] gpgsign`,
the `[filter "lfs"]` block, and the personal `[alias]` entries.

**Caveat (documented in README):** because the repo lives at `~/.config`, the
tracked `git/config` *is* git's XDG global file. `git config --global …` may
write into it rather than `local`. The README steers users to edit `git/local`
(or use `git config --file ~/.config/git/local …`).

## `install.sh` change

After the symlink step and before the zsh hand-off, add:

```sh
# Seed a per-user git identity file (edit it with your name/email).
if [ -f "$CONFIG_DIR/git/local.example" ] && [ ! -f "$CONFIG_DIR/git/local" ]; then
    cp "$CONFIG_DIR/git/local.example" "$CONFIG_DIR/git/local"
fi
```

## README change

Add a short "Set your git identity" subsection under **Getting started**:
edit `~/.config/git/local` with your name and email (the installer creates it
from a template). Mention the `git config --file` form and that git will prompt
until it's set.

## Verification

- **Vendored config files** (`bat/config`, `git/ignore`, `wgetrc`,
  `glow/glow.yml`, `glow/styles/catppuccin-mocha.json`) → `diff` byte-identical
  against `hollow/dotfiles@main`.
- **`zsh/.zshrc`** → strict line-subset: every non-blank line exists in
  upstream's `.zshrc`. `zsh -n zsh/.zshrc` passes.
- **`Brewfile`** → every entry exists in upstream's `Brewfile` except `glow`
  (the sole expected exception); `brew bundle list --file=./Brewfile --all`
  parses.
- **`git/config`** → every line exists in upstream's `git/config` except the
  net-new `[include]` / `path = local` lines.
- **Validity:** `glow/styles/catppuccin-mocha.json` parses as JSON;
  `glow/glow.yml` parses as YAML.
- **Net-new, not checked against upstream:** `git/local.example`,
  `git/.gitignore`, the `[include]` directive, `brew "glow"`, and the
  `install.sh` seed step.
- **Manual smoke test (extended):** `l` (eza, colored), `bat` paging + colored
  man pages, a `you-should-use` reminder, `df` → duf, `glow` renders a markdown
  file with the theme, and `git config user.email` reflects `git/local` once
  set.

## Acceptance criteria

- All Ring 2 tools install via `brew bundle` and their `.zshrc` blocks load
  without error on a fresh shell.
- The faithfulness checks above pass (only the enumerated deviations differ).
- A fresh install seeds `git/local`; editing it sets the user's git identity;
  an unedited `git/local` (or missing one) leaves git prompting normally.
- `LICENSE` and the Ring 1 files remain unchanged except the `Brewfile`,
  `zsh/.zshrc`, `install.sh`, and `README.md` edits described here.
