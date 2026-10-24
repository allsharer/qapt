# qapt — Total Package Installer / Uninstaller

`qapt` wraps `apt` to track every package (including pulled-in dependencies) under a named **label**. When you remove a label, everything it installed is purged — except packages that other labels or other installed software still need, or that you have since installed explicitly. The removal plan is verified with an apt simulation and shown to you first, so nothing outside the label is ever removed.

Accepts both repo package names and paths to local `.deb` files (e.g. something you downloaded manually) — both are installed and tracked the same way.

## Installation

Copy `qapt` to somewhere on your `PATH` and make it executable:

```bash
sudo cp qapt /usr/local/bin/qapt
sudo chmod +x /usr/local/bin/qapt
```

## Usage

```
qapt install PACKAGE|./FILE.deb [PACKAGE|./FILE.deb ...] [--label NAME] [--desc "description"] [--each]
qapt update  PACKAGE|./FILE.deb [--label NAME]
qapt remove  LABEL [--dry-run]
qapt list
qapt history
qapt audit   LABEL
qapt --help
```

## Commands

| Command | Description |
|---------|-------------|
| `install` | Install packages and track them under a label |
| `update` | Update a single already-tracked app to a newer version (repo package or local `.deb`) |
| `remove` | Purge everything under a label that nothing else needs (`--dry-run` shows the plan only) |
| `list` | Show all active labels and their top-level packages |
| `history` | Show labels that have been removed |
| `audit` | Show what is tracked under a label without removing anything |

## Install options

| Option | Description |
|--------|-------------|
| `--label NAME` | Group packages under this label (defaults to package name for single installs) |
| `--desc TEXT` | Free-text description stored with the label |
| `--each` | Install each package under its own auto-label without prompting |

## Examples

```bash
# Single package — label is auto-set to "gimp"
qapt install gimp

# Multiple packages under one label
qapt install gimp inkscape --label graphics_suite --desc "Design tools"

# Multiple packages each tracked independently (no prompting)
qapt install gimp inkscape vlc --each

# Install a downloaded .deb — label auto-set from the package's own name
qapt install ./google-chrome-stable_current_amd64.deb

# Mix a repo package and a local .deb under one label
qapt install gimp ./slack-desktop_amd64.deb --label extras

# Update a single already-tracked app to a newer version
qapt update ./qbible_1.2.0_all.deb
qapt update firefox

# Preview what removing a label would do, then remove it
qapt remove gimp --dry-run
qapt remove gimp

# See all tracked labels
qapt list

# Inspect a specific label
qapt audit gimp

# Review previously removed labels
qapt history
```

## How it works

### Install

1. **Preview** — simulates the install (`apt-get -s`) to learn exactly which packages it will add. If apt would have to *remove* existing packages to make room, they are listed and you must confirm.
2. **Before install** — snapshots the full set of installed packages via `dpkg-query`.
3. **apt-get install** — installs the requested packages normally (waiting up to 5 minutes if another apt process holds the lock). A local `.deb` path is passed straight to `apt-get install`, which resolves its dependencies from your configured repos just like a normal package; its real package name (read from the file's control data) is what gets tracked, not the file path.
4. **After install** — snapshots again and diffs to find every new package (including dependencies). Only packages the preview expected are tracked: anything else that appeared meanwhile (e.g. unattended-upgrades installing a kernel) is reported and left untracked, so a later `qapt remove` can never purge it.
5. **Label file** — stores the top-level package names and the full dependency list under `~/.local/share/qapt/active/`.

If the install fails partway, whatever did get installed is still tracked under the label (marked `PARTIAL INSTALL`), so `qapt remove` can clean it up.

### Remove

Nothing is changed until the plan has been worked out, verified and shown to you:

1. Packages tracked by another active label are **kept**.
2. Dependencies you have since installed explicitly (apt marks them *manual*) are **kept**.
3. For the rest, apt is asked which would become unneeded if the label went away: they are marked "automatic" in a scratch copy of apt's state file and an autoremove is simulated. Anything other installed software still **depends on or recommends** is **kept** (a mere *suggestion* doesn't count).
4. A purge is simulated to confirm nothing outside the label would be removed with it. If something would be, the label packages it depends on are kept too, and the check repeats until the plan is clean.
5. The removal is refused if it would take away your current login screen (display manager).
6. You see what will be removed and what is kept (and why), then confirm. `--dry-run` stops here.
7. Exactly the planned packages are purged (`apt-get purge`, which also removes their system-wide settings). No system-wide `autoremove` is run, so unrelated packages are never touched.
8. Folders in `~/.config`, `~/.local/share`, `~/.cache` and dot-folders in `~` that match the removed packages' names or commands are listed, and you can move each one to the trash (via `trash-put`, if installed) — apt never removes per-user settings.
9. If the original install had removed conflicting packages, the command to restore them is shown.

**Updating a single app:** `qapt install` refuses to reinstall a package that's already present, which is why a newer local `.deb` for an already-tracked app (e.g. `qbible_1.1.0` → `qbible_1.2.0`) needs `qapt update` instead — it re-runs `apt-get install` against the new version/file regardless of what's currently installed, then folds any newly-pulled dependencies into the label's tracked set. `update` only works on a label that tracks exactly one top-level package; a multi-package label has no single "the app" to update, so it must be removed and reinstalled instead.

## Data layout

```
~/.local/share/qapt/
  active/
    <label>.meta    — label name, packages, description, install date,
                      and packages the install replaced (if any)
    <label>.pkgs    — full package list (top-level + dependencies)
  history/
    <label>.meta    — same as active meta, written on removal, plus the
                      removal date and any packages that were kept
```
