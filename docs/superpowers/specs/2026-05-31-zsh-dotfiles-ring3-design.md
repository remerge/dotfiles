# Remerge dotfiles — Ring 3 (bottom, tmux, vim) design

**Date:** 2026-05-31
**Status:** Approved (design); ready for implementation planning
**Repo:** <https://github.com/remerge/dotfiles>
**Builds on:** Ring 0 + Ring 1 + Ring 2 (merged) — see
`docs/superpowers/specs/2026-05-31-zsh-dotfiles-ring2-design.md`
**Upstream pin:** `hollow/dotfiles@main` at commit `9de9bf6`

## Goal

Extend the merged zsh skeleton with three more tools from
<https://github.com/hollow/dotfiles>, staying a faithful **subset** of upstream:
a `diff` against upstream should show only deletions and the previously-trimmed
files. Ring 3 adds a system monitor (`bottom`), a terminal multiplexer (`tmux`),
and an editor (`neovim`), plus the one shell helper `tmux` depends on.

Ring 3 introduces **no intentional deviations** — every kept line is
byte-identical to upstream and every added tool is already in upstream's
`Brewfile`.

## Context: upstream refactor

Between Ring 2 and Ring 3, upstream had a large refactor (its `Brewfile` was
trimmed from ~130 to ~50 entries, and many `bin/*` scripts moved into `zsh/`).
Ring 3 ignores all of that churn and ports only the three tools below plus the
`zsh/clone` helper. The faithfulness checks compare against the pinned commit
`9de9bf6`.

## Scope (decided)

In scope:

- **bottom** (`btm`): a system monitor. Config-only `.zshrc` impact (none) plus a
  vendored `bottom.toml`.
- **tmux**: terminal multiplexer with the upstream `.zshrc` block, the vendored
  `tmux.conf`, and TPM auto-bootstrap via the `zsh/clone` helper.
- **neovim** (invoked as `vim`): the upstream `.zshrc` block plus the vendored
  `vimrc` and the `vim-plug` autoloader.

Out of scope / deferred:

- **mc** (midnight commander): deferred to a later ring. (Upstream ships `mc/`
  config but no `Brewfile` entry; skipped for now.)
- **btop**: not ported — upstream replaced it with `bottom`, which Ring 3 uses.
- **tmux-xpanes**: deferred (its `brew "tmux-xpanes"` entry and the
  `greymd/tmux-xpanes` zi block), like fzf/atuin were deferred in Ring 2.

## Faithfulness principle (carried over)

Every kept line stays byte-identical to upstream. `zsh/.zshrc` remains a strict
line-subset: only upstream lines, re-inserted at their original relative
positions. **Ring 3 adds no deviations** — `bottom`, `tmux`, and `neovim` are
all present in upstream's `Brewfile`.

## File inventory

### Modify

- `Brewfile` — add `bottom`, `neovim`, `tmux` (alphabetical).
- `zsh/.zshrc` — add the tmux and vim blocks (strict subset; see below).

### Create — vendored verbatim from `hollow/dotfiles@9de9bf6`

- `bottom/bottom.toml` — bottom's config (self-contained; the Catppuccin Mocha
  palette is inline, no external theme file).
- `tmux/tmux.conf` — tmux config (declares TPM plugins and bootstraps TPM).
- `vim/vimrc` — neovim/vim config (XDG paths + `vim-plug` + the upstream vimrc).
- `vim/autoload/plug.vim` — the `vim-plug` plugin manager (third-party, vendored
  verbatim as upstream vendors it).
- `zsh/clone` — the helper `:tmux-update` calls to clone TPM. Autoloaded
  function; vendored **non-executable** (`-rw-r-----`), matching the other
  autoloaded helpers (`add`, `has`, `link`, `uri-parse`). Depends only on
  `uri-parse`, which is already byte-identical in this repo.

### Not added

- **bottom** has no `.zshrc` block (upstream has none); it reads
  `~/.config/bottom/bottom.toml` by default.
- **tmux-xpanes** brew entry and zi block (deferred).
- **mc** (deferred): no `mc/` files, no `.zshrc` block, no `Brewfile` entry.

## Path mapping

The repo lives at `~/.config`, so vendored directories map directly:

- `bottom/bottom.toml` → `~/.config/bottom/bottom.toml` (bottom's default path).
- `tmux/tmux.conf` → `~/.config/tmux/tmux.conf` (referenced by
  `ZSH_TMUX_CONFIG`).
- `vim/vimrc` → `~/.config/vim/vimrc` (sourced by `VIMINIT`).
- `vim/autoload/plug.vim` → `~/.config/vim/autoload/plug.vim` (on the vim
  runtimepath set by `vimrc`).
- `zsh/clone` → `~/.config/zsh/clone` (autoloaded; `zsh/` is on `FPATH`).

## `zsh/.zshrc` additions

Both blocks below are byte-identical to upstream and inserted preserving
upstream's relative order. Upstream's order among the blocks this repo already
has is `colored-man-pages → mc → rsync → tmux → tmux/xpanes → vim → wget →
you-should-use`. With mc and xpanes skipped, the kept order is
`colored-man-pages → rsync → tmux → vim → wget`. So **tmux** and then **vim** are
inserted between the existing `rsync` block and the existing `wget` block.

**tmux:**

```zsh
# tmux: a terminal multiplexer
# https://github.com/tmux/tmux
:tmux-load() {
    export TMUX_PLUGIN_MANAGER_PATH="${XDG_CACHE_HOME}/tmux/plugins"
    export ZSH_TMUX_CONFIG="${XDG_CONFIG_HOME}/tmux/tmux.conf"
    export ZSH_TMUX_DEFAULT_SESSION_NAME="default"
    export ZSH_TMUX_FIXTERM="false"
    alias T=tmux
}

:tmux-update() {
    :tmux-load
    clone tmux-plugins/tpm "${TMUX_PLUGIN_MANAGER_PATH}/tpm"
    ${TMUX_PLUGIN_MANAGER_PATH}/tpm/bin/install_plugins
}

zi auto has"tmux" silent for OMZP::tmux
```

`:tmux-load` (the z-a-auto load hook) sets the TPM path, points the OMZP tmux
plugin at the vendored `tmux.conf`, and aliases `T`. `:tmux-update` (the
z-a-auto update hook) clones TPM via the vendored `clone` helper and runs the
TPM plugin install, so the plugins declared in `tmux.conf` install
automatically. The `tmux/xpanes` block is omitted (deferred).

**vim:**

```zsh
# vi improved
# https://github.com/vim/vim
zi auto has"nvim" for neovim
alias vim=nvim
export VIMINIT="set nocp | source ${XDG_CONFIG_HOME}/vim/vimrc"
export EDITOR="${commands[nvim]}"
```

`vim` is aliased to `nvim`; `VIMINIT` sources the XDG `vimrc`; `EDITOR` points at
the `nvim` binary. The vimrc declares plugins via `vim-plug` (e.g.
`tomasiser/vim-code-dark`); these install on demand via `:PlugInstall` — faithful
to upstream, which has no auto-install hook in the vimrc.

## `Brewfile` additions

Add `bottom`, `neovim`, `tmux` (alphabetical). All three exist in upstream's
`Brewfile` at the pinned commit — **no deviations**. (`tmux-xpanes` and
`midnight-commander` are intentionally not added.)

## Verification

- **Vendored verbatim files** (`bottom/bottom.toml`, `tmux/tmux.conf`,
  `vim/vimrc`, `vim/autoload/plug.vim`, `zsh/clone`) → `diff` byte-identical
  against `hollow/dotfiles@9de9bf6`.
- **`zsh/clone`** is **non-executable** (matching `add`/`has`/`link`/`uri-parse`)
  and `zsh -n`-clean. It autoloads and resolves `uri-parse` (already present).
- **`zsh/.zshrc`** → strict line-subset: every non-blank added line exists in
  upstream's `.zshrc`. `zsh -n zsh/.zshrc` passes.
- **`Brewfile`** → every entry exists in upstream's `Brewfile` (no exceptions);
  `brew bundle list --file=./Brewfile --all` parses.
- **Validity:** `bottom/bottom.toml` parses as TOML.
- **Manual smoke test:** `btm` runs with the Catppuccin theme; `T`/`tmux` starts
  and loads the vendored `tmux.conf`, with TPM cloned and plugins installed on
  first update; `vim` launches `nvim`, sources the XDG `vimrc`, and `:PlugInstall`
  installs the declared plugins; `echo $EDITOR` points at `nvim`.

## Acceptance criteria

- `bottom`, `tmux`, and `neovim` install via `brew bundle`, and the tmux/vim
  `.zshrc` blocks load without error on a fresh shell.
- The vendored files are byte-identical to upstream at `9de9bf6`; `zsh/clone`
  autoloads and lets `:tmux-update` bootstrap TPM.
- The faithfulness checks above pass with **no** deviations.
- `LICENSE` and all prior-ring files remain unchanged except the `Brewfile` and
  `zsh/.zshrc` edits described here.
