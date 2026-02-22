# AGENTS.md — screen / tmux config repository

This repository is a [GNU Stow](https://www.gnu.org/software/stow/) dotfiles
package for GNU screen and tmux configuration.  It is stowed into `~` via
`~/.STOW/screenrc/` (or equivalent).  **All edits must be made to files inside
this repository**, not to the stowed paths under `~`.

## Repository layout

```
.screenrc               # top-level GNU screen config (sources .screen.d/COMMON)
.screen.d/              # modular GNU screen config snippets
.tmux.conf              # top-level tmux config (sources .tmux.d/* files)
.tmux.d/                # modular tmux config snippets
  tpm                   # TPM plugin list + plugin-specific config sourcing
  key-bindings          # prefix key, vi/emacs mode-keys, misc bindings
  navigation            # pane/window movement bindings
  prompt-navigation     # in-copy-mode search prompt sequences
  status-bar            # colours, hostname, session name display
  options               # terminal type, history limit, clipboard, etc.
  mouse                 # mouse bindings (legacy, mostly commented out)
  fzf                   # tmux-fzf plugin config
  fingers               # tmux-fingers plugin config
  extrakto              # extrakto plugin config
  standard-session      # session startup helper (currently disabled)
  lib                   # bash helper functions for session setup scripts
  actions/              # hook scripts invoked by bin/tmux-action (sourced in order)
    10-URLs             # open http/ftp URLs
    20-local-files      # open file:line references in $EDITOR via ef
    30-blockchain       # EVM address/tx hash → block explorer menu
    99-fallback         # Google / DuckDuckGo search menu
.terminfo/              # custom terminfo entries (compiled via .cfg-post.d/screenrc)
.cfg-post.d/screenrc    # post-stow script: compile terminfo entries
bin/
  tmux-action           # dispatches a selected string through actions/ hooks
  tmux-current-session  # prints the current tmux session name
tmp/                    # scratch files (not stowed); safe to write here
```

## Stow / deployment

```bash
# Stow (deploy) this package
stow -d ~/.STOW -t ~ screenrc

# Re-stow after adding new files
stow -R -d ~/.STOW -t ~ screenrc
```

The stow target symlinks **directories** wholesale when there is no conflict,
so `~/.tmux.d` is itself a symlink to `.STOW/screenrc/.tmux.d`.  Individual
files inside are **not** separately symlinked.

## Applying config changes

```bash
# Reload tmux config (run inside tmux)
tmux source ~/.tmux.conf

# Or via the default prefix binding
<prefix> r          # if bound — check key-bindings first

# Reload GNU screen config
:source ~/.screenrc
```

There are no build steps, compiled outputs, or test suites in this repo.

## TPM plugins

Managed via [tpm](https://github.com/tmux-plugins/tpm).  Plugin list is in
`.tmux.d/tpm`.  After editing:

```
<prefix> I    # install new plugins
<prefix> U    # update plugins
<prefix> M-u  # remove unlisted plugins
```

Plugins live in `~/.tmux/plugins/` (not stowed, not in this repo).

When a plugin ships a binary (e.g. `tmux-fingers` v2 uses
`~/.tmux/plugins/tmux-fingers/bin/tmux-fingers`), always reference the
**current binary path** rather than old script paths like `scripts/*.sh`.
Check `~/.tmux/plugins/<plugin>/` before writing any `run-shell` invocations.

## Code style

### tmux config files (`.tmux.conf`, `.tmux.d/*`)

- Indent with spaces (2 spaces per level where indentation applies).
- Align option values and binding targets where it aids readability.
- No trailing whitespace, including on blank lines.
- Group related bindings together; separate groups with a blank line.
- Use `bind -n` for bindings that don't require the prefix.
- Use `run-shell -b` (background) for commands that should not block tmux.
- Prefer `#{pane_id}` over `#{session_id}` when targeting the current pane.
- Comments explain *why*, not *what* the option does.
- Keep `source ~/.tmux.d/<file>` calls in `.tmux.conf` / `tpm`; do not
  inline large blocks of config directly in `.tmux.conf`.

### GNU screen config files (`.screenrc`, `.screen.d/*`)

- First line of files intended as shell scripts: `# -*- mode: sh -*-`.
- Follow the same "why not what" comment convention.

### Shell scripts (`bin/`, `.cfg-post.d/`, `actions/`)

- Shebang: `#!/bin/bash` (bash, not sh — associative arrays and `[[` are used).
- Action hook files (`.tmux.d/actions/*`) are **sourced**, not executed;
  they must not define a shebang and must use `# -*- mode: sh -*-` as line 1.
- Use `[[ ... ]]` for conditionals, not `[ ... ]`.
- Use `${BASH_REMATCH[n]}` for regex captures.
- Prefer `local` for function-scoped variables.
- `exit 0` at the end of a matched action hook; subsequent hooks are skipped.
- Helper functions (`urlAction`, `googleSearchAction`, etc.) are defined in
  `bin/tmux-action` and available to all action hooks because hooks are
  sourced after those definitions.
- No trailing whitespace on blank lines.

## Key conventions

- Prefix key is `C-\` (primary) and `F9` (secondary); never rebind `C-b`.
- Mode keys: `vi` for copy mode, `emacs` for the command prompt.
- Plugin trigger bindings are documented with a comment showing how they were
  discovered, e.g. `tmux list-keys G <plugin-name>`.
- VPS-specific overrides (e.g. red status bar) use `if-shell 'hostname -s |
  grep -q "^vps"'` guards.

## Common pitfalls

- **Edit repo files, not stowed paths.** The stowed `~/.tmux.d` is a symlink
  to `.STOW/screenrc/.tmux.d`; individual files inside it (e.g.
  `~/.tmux.d/fingers`) therefore appear as plain files but actually resolve
  outside this repo.  Editing them triggers out-of-repo permission prompts and
  does not update git history.  Always edit under
  `/home/ubuntu/.GIT/adamspiers.org/screen/`, never via `~/.tmux.d/`,
  `~/.tmux.conf`, or any other stowed path.
- **Plugin API changes.** tmux-fingers v2 dropped `scripts/tmux-fingers.sh`;
  use `bin/tmux-fingers start '#{pane_id}'` instead.  Verify the current
  plugin layout before writing `run-shell` lines.
- **`.tmux.conf` is not re-read on `tmux new-session`** if the server is
  already running.  Use `tmux source ~/.tmux.conf` or `<prefix> r`.
- **`standard-session` is intentionally disabled** in `.tmux.conf` (causes
  "can't establish current session" errors on server startup).
- **terminfo entries** in `.terminfo/` are compiled by `.cfg-post.d/screenrc`
  after stowing; if you add new entries you must re-run that script manually.
