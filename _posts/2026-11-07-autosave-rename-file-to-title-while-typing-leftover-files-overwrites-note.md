---
title: "Autosave renames the file to the note's title: leftover files, and a note silently overwritten (os.rename, exit 0)"
description: >-
  A notes app that names each file after the note's first line, and renames it
  on every autosave, left a stray shop.md next to shopping-list.md and replaced
  an existing todo.md with a new note called "Todo". There was no error. On
  macOS, os.rename and mv replace an existing file without asking. Reproduced
  with a small Python script, then fixed by keeping saving and naming apart:
  a save only writes to the path the note already has, and the rename happens
  once, when the note is closed, into a name that is checked to be free.
tags: [python, macos, files, autosave]
---

You start a new note, type `Todo` as the title, and your older note called
`Todo`, with the list underneath it, is gone. **Or** you type `Shopping list`
and the folder now has both `shop.md` and `shopping-list.md`. Nothing printed
an error, and every save returned success.

The app does the obvious thing: the file name comes from the note's first line,
so each autosave works out the name again and renames the file if the title
changed. That puts two jobs in one step, saving the text and choosing a
name, and both cases above come from that:

- **The rename replaces whatever already has that name.** On macOS
  `os.rename` (and `mv`) replace an existing file without asking.
- **The editor doesn't learn the new name until the save replies.** A save sent
  before then still carries the old path, or none, and creates a second file.

I reproduced both on my Mac with a short Python module, then split saving from
naming.

## The autosave that names the file

```python
# notes.py — autosave that names the file after the note's first line
import os, re, threading

lock = threading.Lock()

def name_for(folder, text):
    lines = text.strip().splitlines()
    first = lines[0] if lines else ""
    slug = re.sub(r"[^a-z0-9]+", "-", first.lower()).strip("-")[:40]
    return os.path.join(folder, (slug or "untitled") + ".md")

def save(folder, path, text):
    """Write text; if the first line changed, rename the file to match. Returns the path."""
    with lock:
        new = name_for(folder, text)
        if path and path != new and os.path.exists(path):
            os.rename(path, new)
        with open(new, "w") as f:
            f.write(text)
        return new
```

The lock means only one save runs at a time, so neither bug below is two saves
writing to one file at the same moment.

To follow along, put this and the scripts below in an empty folder and run
the commands from there.

## Case 1: a second note with the same title replaces the first

Note A is titled `Todo`. Then a new note B is started, and its title is typed
as `Tod`, then `Todo`:

```python
# collide.py — a second note gets the same title as an existing one
import os, sys
from notes import save

folder = sys.argv[1]
a = save(folder, None, "Todo\nrenew passport\npay rent\n")
b = save(folder, None, "Tod\n")            # new note, user is still typing the title
b = save(folder, b, "Todo\n")               # ...and finishes it
print("note A path:", a)
print("note B path:", b)
for name in sorted(os.listdir(folder)):
    print(name, repr(open(os.path.join(folder, name)).read()))
```

```console
$ mkdir one && python3 collide.py one; echo "exit $?"
note A path: one/todo.md
note B path: one/todo.md
todo.md 'Todo\n'
exit 0
```

One file is left. A's text (`renew passport`, `pay rent`) is gone, and both
notes now point at the same path. The rename from `tod.md` to `todo.md`
replaced A's file. If B had been typed straight to `Todo` with no save in
between, `open(new, "w")` would have done the same.

It's not just Python. `os.rename` and `mv` both replace the target here:

```console
$ mkdir two && cd two
$ echo "renew passport" > todo.md; echo "Todo" > tod.md
$ python3 -c 'import os; os.rename("tod.md", "todo.md")'; echo "exit $?"
exit 0
$ cat todo.md
Todo
$ echo "renew passport" > todo.md; echo "Todo" > tod.md
$ mv tod.md todo.md; echo "exit $?"
exit 0
$ cat todo.md
Todo
$ cd ..
```

`mv -n` refuses, but it still exits 0, so a script can't tell:

```console
$ cd two
$ echo "renew passport" > todo.md; echo "Todo" > tod.md
$ mv -n tod.md todo.md; echo "exit $?"
exit 0
$ cat todo.md
renew passport
$ ls
tod.md
todo.md
$ cd ..
```

## Case 2: a save sent before the last reply leaves a stray file

Here the editor sends each autosave without waiting for the one before, the
way a `fetch` from a debounced keystroke handler does. The save takes a moment
each way (network, sync, a slow disk; here a `sleep`). The editor only learns
the file's new name when a reply comes back:

```python
# typing.py — an editor that autosaves while you type, without waiting for replies
import os, sys, threading, time
from notes import save

folder = sys.argv[1]
TRIP = 0.3             # seconds each way between editor and disk (network, sync, ...)
path = None            # where the editor thinks the note lives

def autosave(text):
    sent_path = path                    # what the editor knows when it sends
    def request():
        global path
        time.sleep(TRIP)
        result = save(folder, sent_path, text)
        time.sleep(TRIP)
        path = result                   # reply arrives: editor remembers the new name
    t = threading.Thread(target=request)
    t.start()
    return t

# (seconds since the previous save, text in the editor)
typing = [(0.0, "Shop"),
          (0.2, "Shopping list"),               # sent before the first reply
          (1.0, "Shopping list\nmilk\neggs")]   # sent after both replies
threads = []
for pause, text in typing:
    time.sleep(pause)
    threads.append(autosave(text))
for t in threads:
    t.join()

print("editor thinks the note is:", path)
for name in sorted(os.listdir(folder)):
    print(name, repr(open(os.path.join(folder, name)).read()))
```

```console
$ mkdir three && python3 typing.py three; echo "exit $?"
editor thinks the note is: three/shopping-list.md
shop.md 'Shop'
shopping-list.md 'Shopping list\nmilk\neggs'
exit 0
```

The first save created `shop.md`. The second was sent 0.2 s later, before the
first reply, so it still had no path and created `shopping-list.md` as a new
file. Nothing ever comes back for `shop.md`. In a notes list it shows up as a
second note with a few letters of the title in it.

This depends on timing: the stray file appears when a save is sent before the
previous reply is back. My first version used `TRIP = 0.15` and a 0.1 s gap,
and one run in five came out differently (only `shopping-list.md` left). I
didn't trace that run. The sleeps are not exact on this Mac either: in a
thread, `time.sleep(0.3)` took about 0.45 s. So the numbers in the script are
nominal, and I picked them far enough apart that every run I did printed the
output above.

## Fix: saving writes, naming is a separate step

Two rules:

1. **A save only writes to the path the note already has.** It never picks or
   changes a name. A new note gets its file (and path) *before* the first key,
   so every save, even one sent before the last reply, has the right path.
2. **The name is changed once, when no save is in flight**: when the note is
   closed or the user leaves it. The new name has to be one that doesn't exist yet.

```python
# fixed.py — saving never picks a name; naming is a separate step that refuses to overwrite
import os, threading
from notes import name_for

lock = threading.Lock()

def unique(folder, base):
    """Create an empty file named base.md, base-2.md, ... that doesn't exist yet."""
    n = 1
    while True:
        path = os.path.join(folder, base + (".md" if n == 1 else "-%d.md" % n))
        try:
            with open(path, "x"):              # "x": fail if it already exists
                return path
        except FileExistsError:
            n += 1

def new_note(folder):
    with lock:
        return unique(folder, "untitled")

def save(path, text):
    """Autosave: write to the path the note already has. Never renames."""
    with lock:
        if not path or not os.path.exists(path):
            raise FileNotFoundError(path)      # don't silently create a second file
        with open(path, "w") as f:
            f.write(text)
        return path

def name_after_title(path):
    """Call when the editor closes the note (no save in flight). Returns the new path."""
    with lock:
        folder = os.path.dirname(path)
        text = open(path).read()
        want = name_for(folder, text)
        if want == path or not text.strip():   # same name, or no title yet: keep it
            return path
        base = os.path.basename(want)[:-3]
        new = unique(folder, base)             # reserve a free name, never an existing one
        os.replace(path, new)                  # replaces only the empty file we just made
        return new
```

`open(path, "x")` is the part that checks: it creates the file only if the
name is free, and raises `FileExistsError` if it isn't. There's no gap between
checking and creating where another save could take the name. The rename then
lands on the empty file it just reserved, never on another note.

Both cases again, with `fixed.py`:

```python
# check.py — the two cases from above, with fixed.py
import os, sys, threading, time
from fixed import new_note, save, name_after_title

folder = sys.argv[1]

# 1. a second note titled "Todo"
a = new_note(folder)
save(a, "Todo\nrenew passport\npay rent\n")
a = name_after_title(a)
b = new_note(folder)
save(b, "Tod\n")
save(b, "Todo\n")
b = name_after_title(b)

# 2. saves sent before the previous reply came back
TRIP = 0.3
c = new_note(folder)                   # the editor has the path before the first key
def autosave(text):
    def request():
        time.sleep(TRIP); save(c, text); time.sleep(TRIP)
    t = threading.Thread(target=request); t.start(); return t
threads = []
for pause, text in [(0.0, "Shop"), (0.2, "Shopping list"), (1.0, "Shopping list\nmilk\neggs")]:
    time.sleep(pause)
    threads.append(autosave(text))
for t in threads:
    t.join()
c = name_after_title(c)                # closing the note: all saves are done

print("A:", a, " B:", b, " C:", c)
for name in sorted(os.listdir(folder)):
    print(name, repr(open(os.path.join(folder, name)).read()))
```

```console
$ mkdir four && python3 check.py four; echo "exit $?"
A: four/todo.md  B: four/todo-2.md  C: four/shopping-list.md
shopping-list.md 'Shopping list\nmilk\neggs'
todo-2.md 'Todo\n'
todo.md 'Todo\nrenew passport\npay rent\n'
exit 0
```

A keeps its text, B becomes `todo-2.md`, and the shopping note is one file.

### A save to the old path now fails instead of making a copy

After the rename, something may still hold the old path: another tab, or a
save that was queued somewhere. The old `save` would quietly write a new file
there. The fixed one raises:

```python
# stale.py — a save that still has the old path after the rename
import sys
from fixed import new_note, save, name_after_title

folder = sys.argv[1]
p = new_note(folder)
save(p, "Shop\n")
q = name_after_title(p)
print(p, "->", q, flush=True)
save(p, "Shop\nmilk\n")              # still using the old path
```

```console
$ mkdir five && python3 stale.py five; echo "exit $?"
five/untitled.md -> five/shop.md
Traceback (most recent call last):
  File "stale.py", line 10, in <module>
    save(p, "Shop\nmilk\n")              # still using the old path
  File "/Users/me/demo/fixed.py", line 26, in save
    raise FileNotFoundError(path)      # don't silently create a second file
FileNotFoundError: five/untitled.md
exit 1
$ ls five
shop.md
```

What the app does with that error is up to you: ask the editor to reload the
note under its new path, or keep the unsaved text and show it. The point is
that it's visible. This is also why the rename waits until the note is closed:
while the editor is still sending saves, renaming would turn them into errors.

If you can't wait for a close (a sync client renaming underneath you, say),
look the note up by an id that never changes, kept in the file or an index,
instead of by its name. I didn't build that version here.

## Clean up

```console
$ rm -r one two three four five
```

## Checklist

- Stray files with part of a title (`shop.md` next to `shopping-list.md`), or
  a note replaced by a newer one with the same title: suspect an autosave that
  renames the file to match the title.
- On macOS, `os.rename`, `os.replace` and `mv` replace an existing target
  without asking, and exit 0. `mv -n` skips, but also exits 0.
- Don't choose or change the file name inside the save. A save writes to the
  path the note already has, and fails if that path is gone.
- Create the file for a new note before the first keystroke, so the first save
  and every one after it already have its path.
- Rename once, when nothing is being saved (closing the note), into a name you
  reserved with `open(path, "x")`, adding `-2`, `-3`… if it's taken.
- To see the in-flight case, add a delay to the save path and send the next
  save before the reply. On a fast local disk it may never show up.

*Tested on macOS 26.5.2 (Apple silicon) with Python 3.8.2 (`/usr/bin/python3`,
Command Line Tools), zsh 5.9 and `/bin/mv` from file_cmds-479, on APFS. The
editor was a Python script with threads and `sleep`, not a browser, and the
saves were function calls, not HTTP. I didn't test other file systems (SMB,
exFAT, cloud-synced folders), case-insensitive title clashes like `Todo` vs
`TODO` (my slug lowercases them anyway), Linux, or other Python versions.*
