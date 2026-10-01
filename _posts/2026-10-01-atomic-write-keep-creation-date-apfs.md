---
title: "Atomic file writes on macOS without losing the file's creation date"
description: >-
  Writing to a temp file and renaming it is the safe way to save — but on APFS the
  renamed file gets a brand-new creation date (birthtime). A two-step utime trick
  restores it. Tested, with output.
tags: [macos, apfs, python, filesystems]
---

I keep notes as plain Markdown files, and an app shows each note's **created** date
from the file's `birthtime`. Saving had one problem: a process killed halfway
through `write()` leaves a truncated note. On a small machine that runs out of
memory, the kernel does kill processes, and `SIGKILL` gives you no chance to clean up.

The textbook fix is the atomic write: write the new content to a temporary file in
the same directory, then `rename()` it over the original. A rename within one
directory is atomic — readers see the old file or the new file, never half of one.

It worked, and every note in the app suddenly said it was **created today**.

## What rename does to birthtime

```python
import os, time

def show(tag, p):
    st = os.stat(p)
    fmt = lambda t: time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(t))
    print(f"{tag:22s} birth={fmt(st.st_birthtime)}  mtime={fmt(st.st_mtime)}")

p = "note.md"
open(p, "w").write("v1")
old = time.time() - 86400 * 30
os.utime(p, (old, old))          # pretend the note is a month old
show("original", p)

tmp = ".note.md.tmp"
open(tmp, "w").write("v2")
os.replace(tmp, p)               # atomic save
show("after temp+rename", p)
```

```console
original               birth=2026-09-01 09:45:23  mtime=2026-09-01 09:45:23
after temp+rename      birth=2026-10-01 09:45:23  mtime=2026-10-01 09:45:23
```

Of course: the file at that path is now a *different* file — the temp file — and it
was born a moment ago. The old inode, with the old birthtime, is gone.

(Notice also that setting `mtime` a month back in the first step pulled the
`birthtime` back with it. That is the behaviour we'll use.)

## The fix: lower mtime to the old birthtime, then put mtime back

On APFS, when you set a file's modification time to something *earlier* than its
birthtime, the birthtime moves back to match. So:

1. Remember the original file's `st_birthtime` before replacing it.
2. Replace the file atomically.
3. Set `mtime` to the old birthtime — birthtime follows it down.
4. Set `mtime` back to "now" — birthtime stays where it is.

```python
def atomic_write(path: str, data: str) -> None:
    born = os.stat(path).st_birthtime if os.path.exists(path) else None
    tmp = os.path.join(os.path.dirname(path) or ".", "." + os.path.basename(path) + ".tmp")
    with open(tmp, "w") as f:
        f.write(data)
        f.flush()
        os.fsync(f.fileno())
    os.replace(tmp, path)
    if born is not None:
        now = os.stat(path).st_mtime
        os.utime(path, (born, born))   # birthtime drops to `born`
        os.utime(path, (now, now))     # mtime returns; birthtime stays
```

Running the same experiment with these two `utime` calls:

```console
after utime(birth)     birth=2026-09-01 09:45:23  mtime=2026-09-01 09:45:23
after restore mtime    birth=2026-09-01 09:45:23  mtime=2026-10-01 09:45:23
```

Created date preserved, modified date correct, and the save is still atomic.

## Details that bite

- **Same directory.** `rename()` is only atomic within one file system; keep the temp
  file next to the target, not in `/tmp`.
- **Hide the temp file.** A leading dot keeps it out of directory listings and file
  watchers. If the process dies before the rename, the dot-file is left behind —
  sweep `.*.tmp` files on the next start, or they pile up unnoticed.
- **Birthtime can only move backwards this way.** The trick works because the old
  birthtime is always earlier than the new file's. You can't push a birthtime
  forward with `utime`.
- **This is APFS behaviour.** It's what I observe on macOS; don't rely on it on Linux
  file systems, which mostly don't expose birthtime for writing at all.
- **Watchers see two extra events.** The two `utime` calls are metadata changes. If
  another process reacts to every change, debounce it.

*Tested on macOS 26.5, APFS, Python 3.12.*
