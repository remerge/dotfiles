# Remerge dotfiles — Ring 10 (vscode) design

**Date:** 2026-06-05
**Status:** Approved (design); ready for implementation planning
**Repo:** <https://github.com/remerge/dotfiles>
**Builds on:** Rings 0–9 (merged) — see
`docs/superpowers/specs/2026-06-03-zsh-dotfiles-ring9-design.md`
**Upstream pin:** `hollow/dotfiles@main` at commit `cef10b6` (unchanged from Ring 9)

## Goal

Port the **vscode** tool section: the `visual-studio-code` cask, a curated set
of VS Code extensions, the three `vscode/` config files, and the `.zshrc`
section that symlinks the config into VS Code's user directory.

This ring departs from the strict "subset only" Brewfile invariant of prior
rings in two documented ways (see Deviations): a **curated** extension list
(not all 91 upstream entries) that also includes **4 extensions not present
upstream**, and an **emptied `mcp.json`**.

## Scope (decided)

### A. Brewfile

- **`cask "visual-studio-code"`** — inserted after `cask "tailscale-app"`
  (upstream cask order is `…tailscale-app, visual-studio-code, whatsapp`;
  `whatsapp` is un-ported, so VS Code becomes the last cask). Exists upstream →
  no cask deviation.
- **A new `vscode "…"` block** of **37 curated extensions**, alphabetically
  sorted (Homebrew Bundle keeps `vscode` entries sorted), placed after the cask
  block. The exact list is in "Extension list" below.

### B. Config files (`vscode/`)

- **`vscode/settings.json`** — vendored **byte-identical** to upstream
  (mode `100644`).
- **`vscode/keybindings.json`** — vendored **byte-identical** to upstream
  (mode `100644`).
- **`vscode/mcp.json`** — shipped **empty**: exactly

  ```json
  {
  	"servers": {}
  }
  ```

  (tab-indented, trailing newline). This is **not** byte-identical to upstream
  (which registers a Playwright MCP server). The file is still present and
  symlinked so each user has a dotfiles-managed `mcp.json` to add their own
  servers. Documented deviation.

### C. `.zshrc` vscode section

Inserted **byte-identical** to upstream, between the existing `brew` block
(after `zi auto has"dscl" for brew`) and the `# 1password` block. Upstream's
order is `brew → python → uv → argcomplete → vscode → 1password`; `python`,
`uv`, and `argcomplete` are un-ported, so vscode sits directly between our
`brew` and `1password` blocks.

```zsh
# vscode: visual studio code editor
# https://code.visualstudio.com
:vscode-load() {
    if ! has "${HOME}/Library/Application Support/Code/User"; then
        return
    fi

    for i in settings keybindings mcp; do
        link "vscode/${i}.json" "Library/Application Support/Code/User/${i}.json"
    done
}

zi auto has"code" wait for vscode
```

The `mcp` entry stays in the symlink loop (the "empty `mcp.json`, still linked"
decision), so the section is byte-identical to upstream; only `mcp.json`'s
*content* deviates. `:vscode-load` is guarded by `has "…/Code/User"`, so it is a
no-op when VS Code has never run — no error on a fresh shell.

## Extension list

The 37 curated extensions, exactly as approved (source: the repo owner's
`extensions.txt`). **33** exist in upstream's `Brewfile` at `cef10b6`; **4** are
intentional additions (marked `[+]`) that back references already present in the
vendored `settings.json` (Catppuccin theme/icons, mise).

```
aaron-bond.better-comments
anthropic.claude-code
arcanis.vscode-zipfs
bibhasdn.unique-lines
bierner.github-markdown-preview
bierner.markdown-checkbox
bierner.markdown-emoji
bierner.markdown-footnotes
bierner.markdown-preview-github-styles
catppuccin.catppuccin-vsc            [+] not upstream
catppuccin.catppuccin-vsc-icons      [+] not upstream
catppuccin.catppuccin-vsc-pack       [+] not upstream
davidanson.vscode-markdownlint
dotjoshjohnson.xml
ecmel.vscode-html-css
editorconfig.editorconfig
formulahendry.auto-close-tag
formulahendry.auto-complete-tag
formulahendry.auto-rename-tag
grapecity.gc-excelviewer
hverlin.mise-vscode                  [+] not upstream
ibm.output-colorizer
jasonnutter.vscode-codeowners
kaiwood.endwise
marvhen.reflow-markdown
mechatroner.rainbow-csv
mkhl.shfmt
redhat.vscode-yaml
repreng.csv
richie5um2.vscode-sort-json
samuelcolvin.jinjahtml
sharat.vscode-brewfile
sleistner.vscode-fileutils
tamasfe.even-better-toml
timonwong.shellcheck
tomoki1207.pdf
yzhang.markdown-all-in-one
```

Note: `catppuccin.catppuccin-vsc-pack` is an extension pack that bundles
`catppuccin-vsc` + `catppuccin-vsc-icons`; listing all three is mildly redundant
but harmless, and is kept as the owner curated it.

`sleistner.vscode-fileutils` backs the `fileutils.renameFile` keybinding in the
vendored `keybindings.json`.

## Deviations (documented)

This is the first ring whose Brewfile is **not** a strict subset of upstream.
Two intentional deviations, both owner-approved:

1. **Curated extension set with additions.** Of upstream's 91 `vscode` entries,
   33 are ported and 58 are omitted; additionally **4 entries not present
   upstream** (`catppuccin.catppuccin-vsc`, `catppuccin.catppuccin-vsc-icons`,
   `catppuccin.catppuccin-vsc-pack`, `hverlin.mise-vscode`) are added because
   the vendored `settings.json` references the Catppuccin theme/icon themes and
   mise, which upstream never listed.
2. **Emptied `mcp.json`.** Content is `{"servers": {}}` instead of upstream's
   Playwright MCP registration; the file remains present and symlinked.

Everything else stays faithful: the cask and the 33 upstream extensions are
byte-identical upstream lines; `settings.json` and `keybindings.json` are
byte+mode identical to upstream; the `.zshrc` section is byte-identical at its
upstream-relative position.

## Dependency analysis

- **cask `visual-studio-code`** installs the `code` CLI; `zi auto has"code"
  wait for vscode` then runs `:vscode-load`.
- **`:vscode-load`** uses the existing `has` and `link` helpers (present since
  the skeleton ring) and only acts when `~/Library/Application Support/Code/User`
  exists. It symlinks `settings.json`, `keybindings.json`, `mcp.json` from the
  repo into that directory.
- **Extensions** are installed by `brew bundle` via the `code --install-extension`
  path; they have no shell-runtime dependency.
- The 4 added extensions back `settings.json` references; no extension depends on
  an un-ported helper.

## File inventory

### Modify
- `Brewfile` — add `cask "visual-studio-code"` and the 37-entry `vscode` block.
- `zsh/.zshrc` — add the vscode section.

### Create — vendored from `hollow/dotfiles@cef10b6`
- `vscode/settings.json` (byte-identical, mode `100644`).
- `vscode/keybindings.json` (byte-identical, mode `100644`).
- `vscode/mcp.json` (emptied to `{"servers": {}}`, mode `100644` — deviation).

## Path mapping

- `vscode/settings.json`, `vscode/keybindings.json`, `vscode/mcp.json` →
  `~/.config/vscode/*.json`, symlinked by `:vscode-load` into
  `~/Library/Application Support/Code/User/*.json`.
- `cask "visual-studio-code"` + the `vscode "…"` block → installed by
  `brew bundle`.
- The vscode `.zshrc` section lives in `~/.config/zsh/.zshrc` (already linked).

## `Brewfile` additions

- **Cask:** `cask "visual-studio-code"` after `cask "tailscale-app"`.
- **Extensions:** the 37 `vscode "…"` lines (sorted) as a new block after the
  casks.

## Verification

- **Cask + upstream extensions:** every `cask` line and every *upstream* `vscode`
  line is byte-identical to upstream; `comm -23` of our `cask`/upstream-`vscode`
  lines against upstream's is empty. The 4 additions are validated as **exactly**
  the known set (`catppuccin.catppuccin-vsc`, `catppuccin.catppuccin-vsc-icons`,
  `catppuccin.catppuccin-vsc-pack`, `hverlin.mise-vscode`) — no other non-upstream
  entry exists. The full `vscode` block equals the sorted contents of the owner's
  curated list (37 lines).
- **`brew bundle list --file=./Brewfile --all`** parses.
- **Config files:** `settings.json` and `keybindings.json` mode+content identical
  to upstream via `git ls-files -s`. `mcp.json` equals exactly `{"servers": {}}`
  (tab-indented, trailing newline) and contains no `servers` members.
- **`zsh/.zshrc`:** the vscode section diffs clean against upstream's; every
  non-blank line of our `.zshrc` exists in upstream's; `zsh -n zsh/.zshrc` passes.
- **Manual smoke test:** `brew bundle install` installs the cask + 37 extensions;
  on a fresh shell with VS Code's user dir present, `:vscode-load` symlinks the
  three files without error; the Catppuccin theme/icons and mise extension resolve
  the corresponding `settings.json` keys.

## Acceptance criteria

- `Brewfile` has `cask "visual-studio-code"` after `cask "tailscale-app"` and a
  37-line sorted `vscode` block equal to the curated list; the 33 upstream
  entries + cask are byte-identical upstream lines; the only non-upstream
  `vscode` entries are the 4 approved additions; `brew bundle list --all` parses.
- `vscode/settings.json` and `vscode/keybindings.json` are byte+mode identical to
  upstream (`100644`); `vscode/mcp.json` is exactly `{"servers": {}}`.
- The vscode `.zshrc` section is byte-identical to upstream between the `brew` and
  `1password` blocks; `zsh -n zsh/.zshrc` passes; `.zshrc` is otherwise a strict
  line-subset of upstream.
- `LICENSE` and all prior-ring files remain unchanged except the `Brewfile` and
  `zsh/.zshrc` edits described here. The temporary `extensions.txt` at the repo
  root is removed before completion (it is an input artifact, not part of the
  dotfiles).
