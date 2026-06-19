# qapt — Total Package Installer / Uninstaller

`qapt` wraps `apt` to track every package (including pulled-in dependencies) under a named **label**. When you remove a label, only the packages that belong exclusively to it are uninstalled — anything shared with another label is left alone.

## Installation

Copy `qapt` to somewhere on your `PATH` and make it executable:

```bash
sudo cp qapt /usr/local/bin/qapt
sudo chmod +x /usr/local/bin/qapt
```

## Usage

```
qapt install PACKAGE [PACKAGE ...] [--label NAME] [--desc "description"] [--each]
qapt remove  LABEL
qapt list
qapt history
qapt audit   LABEL
qapt --help
```

## Commands

| Command | Description |
|---------|-------------|
| `install` | Install packages and track them under a label |
| `remove` | Remove everything under a label (skips packages shared by other labels) |
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

# Remove everything installed under a label
qapt remove gimp

# See all tracked labels
qapt list

# Inspect a specific label
qapt audit gimp

# Review previously removed labels
qapt history
```

## How it works

1. **Before install** — snapshots the full set of installed packages via `dpkg-query`.
2. **apt-get install** — installs the requested packages normally.
3. **After install** — snapshots again and diffs to find every new package (including dependencies).
4. **Label file** — stores the top-level package names and the full dependency list under `~/.local/share/qapt/active/`.

On removal, `qapt` computes the union of packages owned by all *other* active labels and skips anything in that set, so shared dependencies are never accidentally removed.

## Data layout

```
~/.local/share/qapt/
  active/
    <label>.meta    — label name, packages, description, install date
    <label>.pkgs    — full package list (top-level + dependencies)
  history/
    <label>.meta    — same as active meta, written on removal
```
