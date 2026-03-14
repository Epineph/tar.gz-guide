# Tar Archiving on Linux: Practical Excludes, Symlinks, and Safe Defaults

## Blunt recommendation

For Linux and Unix-like systems, use `tar` when you care about:

- symbolic links
- permissions and ownership
- preserving directory structure
- backing up normal filesystem trees correctly

For most practical cases:

```bash
tar -cvJf archive.tar.xz directory/
```

If you prefer faster compression and decompression:

```bash
tar -cvzf archive.tar.gz directory/
```

In practice:

- use `.tar.gz` when speed matters more
- use `.tar.xz` when smaller archives matter more

---

## Core rule for excludes

Do **not** use one universal exclude list for everything.

The correct exclude set depends on what the archive is for.

Exclude things that are typically:

1. **Ephemeral**
   - temporary files
   - runtime state
   - trash
2. **Reproducible**
   - build output
   - package caches
   - dependency caches
3. **Large and low-value**
   - downloads you do not need
   - disposable media
   - easily recreated artifacts

Do **not** casually exclude:

- configuration you actually use
- scripts
- notes
- research data
- manually curated files
- anything expensive to recreate

---

## 1. Project archives

For a source-code or research project archive, I would often exclude generated
output, caches, and dependency directories.

### Example

```bash
tar -cvJf myproject.tar.xz myproject/ \
  --exclude='.git' \
  --exclude='node_modules' \
  --exclude='dist' \
  --exclude='build' \
  --exclude='target' \
  --exclude='__pycache__' \
  --exclude='*.pyc' \
  --exclude='.venv' \
  --exclude='venv' \
  --exclude='.mypy_cache' \
  --exclude='.pytest_cache' \
  --exclude='.Rproj.user' \
  --exclude='.quarto'
```

### Why exclude these?

- `.git`: version history can be omitted if you only want the working tree
- `node_modules`: huge, reinstallable
- `dist`, `build`, `target`: generated output
- `__pycache__`, `*.pyc`: Python bytecode cache
- `.venv`, `venv`: usually rebuildable virtual environments
- `.mypy_cache`, `.pytest_cache`: tool caches
- `.Rproj.user`: machine-local RStudio metadata
- `.quarto`: generated intermediate output in some workflows

### When this would be wrong

This exclude list is wrong if you explicitly want:

- the Git history
- the exact virtual environment
- the compiled artifacts
- a fully reproducible environment snapshot rather than source-only backup

---

## 2. Home-directory backups

For a home backup, I would usually exclude caches, trash, and sometimes
Downloads, but I would **not** exclude actual configuration by default.

### Example: conservative home backup

```bash
tar -cvJf home-backup.tar.xz "$HOME" \
  --exclude="$HOME/.cache" \
  --exclude="$HOME/.local/share/Trash"
```

### Example: more aggressive home backup

```bash
tar -cvJf home-backup-smaller.tar.xz "$HOME" \
  --exclude="$HOME/.cache" \
  --exclude="$HOME/.local/share/Trash" \
  --exclude="$HOME/Downloads" \
  --exclude="$HOME/.npm" \
  --exclude="$HOME/.cargo/registry" \
  --exclude="$HOME/.cargo/git" \
  --exclude="$HOME/.rustup/downloads" \
  --exclude="$HOME/.cache/pip" \
  --exclude="$HOME/.conda/pkgs"
```

### Why exclude these?

- `.cache`: usually regenerates
- `.local/share/Trash`: obvious low-value content
- `Downloads`: often large and disposable, but this is a policy choice
- `.npm`, `.cargo/registry`, `.cargo/git`, `.rustup/downloads`,
  `.cache/pip`, `.conda/pkgs`: dependency/package caches that can often be
  re-fetched

### When this would be wrong

This is wrong if your `Downloads` directory contains important irreplaceable
material, or if you depend on cached packages for offline restoration.

---

## 3. System backups

Do **not** blindly archive `/` without excluding pseudo-filesystems and volatile
paths.

### Example

```bash
sudo tar -cvJf system-backup.tar.xz / \
  --exclude=/proc \
  --exclude=/sys \
  --exclude=/dev \
  --exclude=/run \
  --exclude=/tmp \
  --exclude=/var/tmp \
  --exclude=/mnt \
  --exclude=/media \
  --exclude=/swapfile \
  --exclude=/lost+found
```

### Often also worth considering

```bash
--exclude=/var/cache/pacman/pkg
```

### Why exclude these?

- `/proc`, `/sys`, `/dev`, `/run`: pseudo-filesystems or dynamic runtime state
- `/tmp`, `/var/tmp`: temporary files
- `/mnt`, `/media`: transient mount points; may recurse into external disks
- `/swapfile`: unnecessary in archive
- `/lost+found`: filesystem recovery directory
- `/var/cache/pacman/pkg`: package cache, often huge and reproducible

### When this would be wrong

Excluding package caches is wrong if part of your recovery strategy is to keep a
local package source for faster or offline reinstallation.

---

## Symbolic links: the safe default

This matters.

By default, `tar` stores the **symbolic link itself**, not the contents of the
file it points to.

That is usually the correct behavior.

### Example

If you have:

```text
mydir/link -> /some/target
```

then this:

```bash
tar -cvJf archive.tar.xz mydir/
```

will normally store `link` as a symlink entry.

### Why this is good

Because it preserves the filesystem structure faithfully.

---

## What to avoid with symlinks

Avoid this unless you explicitly want the target contents:

```bash
tar -cvJhf archive.tar.xz mydir/
```

or:

```bash
tar -cvJf archive.tar.xz mydir/ --dereference
```

### What `-h` / `--dereference` does

It tells `tar` to follow symbolic links and archive the target content instead
of the link object.

### Why that can go wrong

- the target may be broken
- the target may be outside the directory tree you intended to archive
- the target may be very large
- the target may require permissions you do not have
- you may silently archive the wrong data

### Blunt rule

For Linux backups, do **not** use `--dereference` unless you intentionally want
that behavior.

---

## If 7z previously failed on symlinks

That is one reason `tar` is the better default for Linux filesystem trees.

The likely issue was one of these:

1. the tool tried to follow symlink targets
2. the tool did not preserve symlinks as naturally as `tar`
3. a broken or inaccessible target caused failure
4. the target was outside the intended archive tree

With `tar`, the safe default is already sensible: store links as links.

---

## Practical examples

## Example A: source-only project archive

```bash
tar -cvJf thesis-project.tar.xz thesis-project/ \
  --exclude='.git' \
  --exclude='build' \
  --exclude='dist' \
  --exclude='__pycache__' \
  --exclude='.venv' \
  --exclude='.Rproj.user'
```

## Example B: fast project archive with gzip

```bash
tar -cvzf thesis-project.tar.gz thesis-project/ \
  --exclude='.git' \
  --exclude='build' \
  --exclude='dist'
```

## Example C: home configuration backup only

```bash
tar -cvJf home-configs.tar.xz \
  "$HOME/.zshrc" \
  "$HOME/.zprofile" \
  "$HOME/.config" \
  "$HOME/.local/share" \
  --exclude="$HOME/.cache" \
  --exclude="$HOME/.local/share/Trash"
```

## Example D: archive home but keep symlinks as symlinks

```bash
tar -cvJf full-home.tar.xz "$HOME" \
  --exclude="$HOME/.cache" \
  --exclude="$HOME/.local/share/Trash" \
  --exclude="$HOME/Downloads"
```

No special symlink flag is needed here. The default behavior is usually the
correct one.

## Example E: inspect what would be in an archive afterward

```bash
tar -tvJf full-home.tar.xz
```

## Example F: extract elsewhere

```bash
mkdir -p ~/restore-test
tar -xvJf full-home.tar.xz -C ~/restore-test
```

---

## A compact policy you can actually remember

### For projects

Exclude:

- `.git` if history is unnecessary
- build output
- caches
- dependency directories you can recreate

### For home backups

Exclude:

- `.cache`
- Trash
- optionally Downloads
- optionally large package caches

### For system backups

Exclude:

- `/proc`
- `/sys`
- `/dev`
- `/run`
- `/tmp`
- `/var/tmp`
- mount points
- swap files

### For symlinks

- keep the default behavior
- do **not** use `--dereference` unless you mean it

---

## Minimal safe starting points

### Smaller home backup

```bash
tar -cvJf backup-home.tar.xz "$HOME" \
  --exclude="$HOME/.cache" \
  --exclude="$HOME/.local/share/Trash" \
  --exclude="$HOME/Downloads"
```

### Smaller project archive

```bash
tar -cvJf project.tar.xz project/ \
  --exclude='.git' \
  --exclude='node_modules' \
  --exclude='build' \
  --exclude='dist' \
  --exclude='__pycache__'
```

### Faster gzip variant

```bash
tar -cvzf project.tar.gz project/ \
  --exclude='.git' \
  --exclude='node_modules' \
  --exclude='build'
```

---

## Final blunt conclusion

Use `tar` on Linux when filesystem semantics matter.

- use `.tar.gz` for speed
- use `.tar.xz` for smaller archives
- exclude caches, temp data, build output, and trash
- do **not** casually exclude real data or configuration
- let symlinks remain symlinks
- avoid `--dereference` unless you explicitly want target contents
