# PERSONAL_PATCH.md

## Scope

This is a personal Warp fork patch for layout-independent keyboard shortcuts,
based on [warpdotdev/warp#11923](https://github.com/warpdotdev/warp/pull/11923).
It targets KDE Plasma on Wayland. It is not an upstream release and does not
claim that the upstream pull request was merged.

## Patch identity

- Repository: warpdotdev/warp
- Branch: personal/layout-aware-current
- Integration commit: efa7f56bed621e6cc6502c8cc2f5246912b40a06 (efa7f56be)
- Commit subject: Merge layout-aware keyboard shortcuts into current Warp
- Local tag: personal/layout-aware-current-working
- Personal binary: $HOME/.local/opt/warp-layout-aware/warp-oss

The integration commit contains the six layout-aware commits from the original
patch line plus a merge of the current upstream base.

## Behavior

The patch addresses shortcuts such as Ctrl+C or Cmd+C failing when a non-Latin
keyboard layout changes the logical character reported by the physical C key.

It provides three opt-in mechanisms:

1. A physical keybinding in keybindings.yaml, for example
   editor:copy: cmd-Code(KeyC).
2. The Smart layout-aware shortcuts setting:
   keyboard.smart_layout_aware_bindings = true.
3. A Layout-aware toggle in the keybinding editor for eligible letters and digits.

Code(NAME) bindings use W3C UI Events physical-key codes. Smart mode promotes
only alphanumeric bindings (KeyA through KeyZ and Digit0 through Digit9).
Symbols are intentionally not promoted because their physical positions vary
between layouts. IME composition suppresses smart matching. Both opt-in modes
default to off. The upstream PR documents the design and platform caveats.

## Git remotes

Expected local remotes:

~~~text
origin  https://github.com/warpdotdev/warp.git
fork    git@github.com:jimvixx/warp.git
~~~

Use origin for upstream inspection and fetches; use fork for personal branches.
Do not push this branch to origin.

~~~bash
cd "$HOME/Repos/warp"
git remote -v
git status -sb
git log -1 --oneline --decorate
~~~

When publishing is explicitly intended:

~~~bash
git push -u fork personal/layout-aware-current
~~~

Pushing is not part of the update workflow below.

## Build and tests

Use the repository's setup instructions for dependencies:

~~~bash
cd "$HOME/Repos/warp"
./script/bootstrap
~~~

Focused checks:

~~~bash
cargo fmt --all -- --check
cargo test --locked -p warpui_core --lib keymap
cargo build --locked --release --bin warp-oss --features gui
~~~

For the broader repository check:

~~~bash
./script/presubmit
~~~

The focused keymap test is the minimum regression check, not a substitute for
manual testing in the target desktop session.

## Local installation on KDE Plasma Wayland

This is a direct local binary installation, not a system package installation.
Keep it in a dedicated directory:

~~~bash
cd "$HOME/Repos/warp"
cargo build --locked --release --bin warp-oss --features gui
install -Dm755 target/release/warp-oss \
  "$HOME/.local/opt/warp-layout-aware/warp-oss"
"$HOME/.local/opt/warp-layout-aware/warp-oss"
~~~

If useful, record the session before testing:

~~~bash
printf 'desktop=%s\nsession=%s\n' "$XDG_CURRENT_DESKTOP" "$XDG_SESSION_TYPE"
~~~

Test with the actual Latin and non-Latin layouts used on KDE Plasma Wayland.
Do not overwrite a vendor installation with an unrelated debug or differently
signed binary.

## Manual validation

1. Open Settings and locate the Keys or keyboard feature settings.
2. Confirm Smart layout-aware shortcuts is off unless intentionally enabled.
3. Enable Layout-aware for one eligible letter binding, or add a test
   Code(KeyC) binding in the user keybindings configuration.
4. Switch between Latin and non-Latin layouts.
5. Verify the selected shortcut with the same physical key under both layouts.
6. If smart mode is enabled, test an alphanumeric shortcut and confirm that a
   symbol shortcut is not silently promoted.
7. If relevant, verify that IME composition does not trigger smart shortcuts.

Passing unit tests does not prove that the compositor, input method, keyboard
layout, and running desktop session deliver the expected physical-key event.

## Manual update workflow

The helper is $HOME/.local/bin/warp-layout-aware-update. In the current checkout
it is mode 664 rather than executable, so invoke it through Bash unless its
permissions are intentionally changed.

~~~text
main repository:   $HOME/Repos/warp
update worktree:   $HOME/Repos/warp-layout-aware-update
base branch:       personal/layout-aware-current
update branch:     personal/layout-aware-next
binary directory:  $HOME/.local/opt/warp-layout-aware
~~~

### Prepare

Start with a clean main worktree on the base branch:

~~~bash
cd "$HOME/Repos/warp"
git switch personal/layout-aware-current
git status -sb
bash "$HOME/.local/bin/warp-layout-aware-update" prepare
~~~

Prepare verifies the base branch and fetches origin. If origin/master is already
an ancestor of the base, it exits without creating an update worktree. Otherwise
it creates personal/layout-aware-next at the update worktree, merges origin/master,
runs formatting, the locked keymap tests, and the locked release build, then writes:

~~~text
$HOME/.local/opt/warp-layout-aware/warp-oss.candidate
$HOME/.local/opt/warp-layout-aware/warp-oss.candidate.commit
~~~

If the merge conflicts, resolve and commit in the update worktree, then run build.
Do not activate an unreviewed or untested candidate.

### Build or rebuild the candidate

~~~bash
bash "$HOME/.local/bin/warp-layout-aware-update" build
~~~

Build requires a clean update worktree and replaces only the candidate and commit
marker. It does not replace the installed warp-oss.

### Review and manually test

~~~bash
git -C "$HOME/Repos/warp-layout-aware-update" status -sb
git -C "$HOME/Repos/warp-layout-aware-update" diff \
  personal/layout-aware-current...personal/layout-aware-next
git -C "$HOME/Repos/warp-layout-aware-update" log -1 --oneline --decorate
cat "$HOME/.local/opt/warp-layout-aware/warp-oss.candidate.commit"
"$HOME/.local/opt/warp-layout-aware/warp-oss.candidate"
~~~

### Activate only after approval

Activation is the state-changing stage:

~~~bash
bash "$HOME/.local/bin/warp-layout-aware-update" activate
~~~

It requires a clean main worktree, verifies the candidate commit, saves the
installed binary as warp-oss.previous, fast-forwards the base branch, atomically
replaces warp-oss, removes the update worktree, and deletes the temporary branch.
It is not part of prepare/build testing.

To restore only the executable after recording a failure:

~~~bash
cp -p "$HOME/.local/opt/warp-layout-aware/warp-oss.previous" \
  "$HOME/.local/opt/warp-layout-aware/warp-oss"
~~~

This does not undo the Git fast-forward.

## Current local evidence

The checkout was inspected while preparing this document:

- personal/layout-aware-current pointed to efa7f56be and had no unstaged changes.
- origin/master was already an ancestor of the base branch.
- Before prepare, no personal/layout-aware-next branch or update worktree existed.
- bash -n ~/.local/bin/warp-layout-aware-update passed.
- git diff --check passed.

The subsequent non-activating prepare run fetched origin/master at 4ff5753e2,
created personal/layout-aware-next, merged without conflicts, passed the focused
keymap test with 29 tests passed, and completed the locked release build. The
candidate commit is 893198da3 and its metadata matches the candidate binary.
The release build emitted existing unused-import, unused-variable, unused-mut,
and dead-code warnings, but no build error. The installed binary was not
replaced, and activate was not run. The candidate also stayed running for a
15-second foreground smoke test in the KDE Plasma Wayland environment
(DISPLAY=:0, WAYLAND_DISPLAY=wayland-0); timeout then stopped it with exit code
124. The candidate was then launched manually by the user in KDE Plasma Wayland,
and the layout-aware shortcuts were reported to work as expected. This is a
manual user-reported PASS; no automated screenshot or input trace was collected.

## Credits and licensing

- Original layout-aware implementation: Desko77 (Dmitriy Lim), in
  [Warp PR #11923](https://github.com/warpdotdev/warp/pull/11923).
- Related upstream issue: [Warp #7026](https://github.com/warpdotdev/warp/issues/7026).
- Local integration: personal/layout-aware-current, including efa7f56be.
- Project: [Warp](https://github.com/warpdotdev/warp), preserving its existing
  license and attribution files.

Warp's README states that warpui_core and warpui are MIT-licensed and the rest
of the repository is AGPLv3-licensed. Keep both license files and existing
notices when redistributing a personal build.
