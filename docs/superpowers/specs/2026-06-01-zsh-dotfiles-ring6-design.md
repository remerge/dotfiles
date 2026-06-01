# Remerge dotfiles — Ring 6 (mise, age, sops) design

**Date:** 2026-06-01
**Status:** Approved (design); ready for implementation planning
**Repo:** <https://github.com/remerge/dotfiles>
**Builds on:** Rings 0–5 (merged) — see
`docs/superpowers/specs/2026-06-01-zsh-dotfiles-ring5-design.md`
**Upstream pin:** `hollow/dotfiles@main` at commit `1c3018c`

## Goal

Port upstream's **`mise`**, **`age`**, and **`sops`** sections, staying a faithful
**subset** of <https://github.com/hollow/dotfiles>: a `diff` against upstream
shows only deletions and the previously-trimmed files.

`mise` is a runtime/tool manager, `age` is a file-encryption tool, and `sops`
ties them together for encrypted-secret editing. They belong in one ring because
`age` and `sops` are config-coupled (both point at the same age key file) and
`mise` declares the same key file via its built-in sops integration.

## Scope (decided)

In scope:

- **mise**: `brew "mise"`, the mise `.zshrc` block (`MISE_SOPS_AGE_KEY_FILE`
  export, `:mise-load` running `mise activate zsh`, lazy `zi auto has"mise"`),
  and a vendored `mise/config.toml` with an **empty `[tools]` table**.
- **age**: `brew "age"` only. No `.zshrc` block, no config — `age` is surfaced
  by the `*_AGE_KEY_FILE` variables in the mise and sops blocks.
- **sops**: `brew "sops"` plus the sops `.zshrc` block (`SOPS_AGE_KEY_FILE`
  export). No vendored config file (see deviations).

Out of scope / deferred:

- The mise-managed **global toolchain** (`bun`, `markdownlint-cli2`, `node`,
  `opentofu`, `prettier`, `yarn`): omitted by emptying `[tools]`. Each tool lands
  in its own future ring.
- **`opentofu`'s** own `.zshrc` section (`TF_PLUGIN_CACHE_DIR`) and every other
  unrelated tool section: not part of this ring.

## Faithfulness principle (carried over)

Every kept line stays byte-identical to upstream. `zsh/.zshrc` remains a strict
line-subset: only upstream lines, re-inserted at their original relative
positions. Ring 6 introduces **two deviations**, both deletions consistent with
the subset principle:

1. **`mise/config.toml`** is vendored with the six `[tools]` entries removed,
   leaving only the `[tools]` header — mise installs and activates but manages no
   global tools yet.
2. **`sops/.gitignore`** (which upstream still has at `1c3018c`) is **not
   vendored**. Upstream will drop this file shortly; once it does and the pin is
   bumped, the omission stops being a deviation.

All three brews — `age`, `mise`, `sops` — exist in upstream's `Brewfile` at
`1c3018c` (`sops` was just added upstream), so there are **no Brewfile
deviations**.

## File inventory

### Modify

- `Brewfile` — add `brew "age"`, `brew "mise"`, `brew "sops"`.
- `zsh/.zshrc` — add the **mise** and **sops** blocks (strict subset).

### Create

- `mise/config.toml` — upstream's, with the `[tools]` entries removed:

  ```toml
  [tools]
  ```

### Not added

- **`sops/.gitignore`** — intentionally omitted (see deviations).
- **age** needs no vendored config; the key file (`~/.config/sops/age/keys.txt`)
  is user runtime state and is never created or tracked by this ring.

## Path mapping

The repo lives at `~/.config`, so vendored directories map directly:

- `mise/config.toml` → `~/.config/mise/config.toml` — mise's global config path
  on macOS (`$XDG_CONFIG_HOME/mise/config.toml`). With an empty `[tools]`, mise
  has a valid global config that declares no tools.
- The age key file referenced by both blocks
  (`${XDG_CONFIG_HOME}/sops/age/keys.txt`) is **not** created by this ring; the
  user supplies it. Nothing under `sops/` is tracked.

## `zsh/.zshrc` additions

Both blocks are byte-identical to upstream and inserted preserving upstream's
relative order. Upstream order in this region is
`ncdu → mise → … → rsync → … → sops → ssh → … → tmux`.

**mise** — inserted between the existing `ncdu` block (after
`link ncduignore .ncduignore`) and the existing `# rsync` block:

```zsh
# mise: dev tools, env vars, task runner
# https://github.com/jdx/mise
export MISE_SOPS_AGE_KEY_FILE="${XDG_CONFIG_HOME}/sops/age/keys.txt"

:mise-load() {
    local _mise_cmd_not_found
    eval "$(mise activate zsh)"
}

zi auto has"mise" for jdx/mise
```

`MISE_SOPS_AGE_KEY_FILE` points mise's built-in sops integration at the age key;
`:mise-load` (a z-a-eval cached hook) activates mise for the shell; the
`zi auto has"mise" …` line loads it only when the `mise` binary is present.

**sops** — inserted between the existing `rsync` block (after
`zi auto wait for OMZP::rsync`) and the existing `# ssh` block (Ring 5):

```zsh
# sops: editor of encrypted files (age, gpg, cloud KMS)
# https://github.com/getsops/sops
export SOPS_AGE_KEY_FILE="${XDG_CONFIG_HOME}/sops/age/keys.txt"
```

`SOPS_AGE_KEY_FILE` lets a standalone `sops` invocation find the same age key
that mise uses.

## `Brewfile` additions

- **Brews:** add `age` (alphabetically first, before `atool`), `mise` (between
  `make` and `ncdu`), and `sops` (between `rsync` and `sponge`).
- No cask changes.

All three exist in upstream's `Brewfile` at `1c3018c` — **no deviations**.

## Verification

- **`mise/config.toml`** → equals upstream's with exactly the six `[tools]`
  entry lines removed (only the `[tools]` header remains); the file is valid TOML.
- **`zsh/.zshrc`** → strict line-subset: every non-blank added line exists in
  upstream's `.zshrc`. `zsh -n zsh/.zshrc` passes.
- **`Brewfile`** → `age`, `mise`, `sops` all exist in upstream's `Brewfile`;
  `brew bundle list --file=./Brewfile --all` parses.
- **Faithfulness audit:** `comm -23` of our `brew`/`cask` lines against
  upstream's is empty (no deviations); the only file-level deviations are the
  emptied `mise/config.toml` `[tools]` and the omitted `sops/.gitignore`.
- **Manual smoke test:** `brew bundle install` installs `age`, `mise`, `sops`;
  on a fresh shell the mise and sops blocks load without error; `mise` resolves
  and `mise activate` runs; `echo $SOPS_AGE_KEY_FILE` and
  `echo $MISE_SOPS_AGE_KEY_FILE` both print `~/.config/sops/age/keys.txt`;
  `mise ls` shows no global tools.

## Acceptance criteria

- `age`, `mise`, and `sops` install via `brew bundle`, and the mise and sops
  `.zshrc` blocks load without error on a fresh shell.
- `mise/config.toml` equals upstream's minus the `[tools]` entries; no
  `sops/.gitignore` is added.
- The faithfulness checks above pass with **no Brewfile deviations**.
- `LICENSE` and all prior-ring files remain unchanged except the `Brewfile` and
  `zsh/.zshrc` edits described here.
