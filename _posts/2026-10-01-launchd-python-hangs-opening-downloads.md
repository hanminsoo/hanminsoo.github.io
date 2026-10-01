---
title: "A launchd job hung for hours opening ~/Downloads (macOS privacy prompt)"
description: >-
  A Python script started by a LaunchAgent froze inside opendir() on ~/Downloads.
  It was waiting for a macOS privacy (TCC) prompt nobody could see. How to spot it
  with `sample`, grant access once, and make the script fail fast next time.
tags: [macos, launchd, tcc, python]
---

A scheduled job on my Mac mini downloads an image in Chrome and then a small Python
script moves the newest file out of `~/Downloads`. Run by hand, it took a second.
Run from a LaunchAgent at 9 a.m., it sat there **for seven hours** doing nothing.

No error, no CPU, no log line. The script had a 90-second retry loop, so it should
have given up long before.

## Look at where it is stuck, not at the code

When a process is idle and "should" have timed out, ask the kernel where it is
blocked. macOS ships `sample`:

```console
$ sample <pid> 1
...
  783 __opendir2  (in libsystem_c.dylib) + 44
    783 open$NOCANCEL  (in libsystem_kernel.dylib) + 64
      783 __open_nocancel  (in libsystem_kernel.dylib) + 8
```

Every sample is inside `opendir()` → `open()`. My retry loop never ran a second time
because the *first* directory listing never returned. Python's
`pathlib.Path.glob()` was simply waiting on the kernel.

## Why opening a folder can block forever

`~/Downloads`, `~/Desktop` and `~/Documents` are protected by macOS privacy
controls (TCC). The first time a program you haven't approved touches them, macOS
asks the user:

> "Python" would like to access files in your Downloads folder.

When you run the script from Terminal, Terminal already has that permission, so you
never see the question. When **launchd** starts the script, the request is attributed
to a different process chain, the prompt is raised again — and the `open()` call
waits for an answer. If nobody is at the screen (that's the point of a scheduled
job), it waits indefinitely.

## Fix it once: grant the permission

Either click **Allow** on the prompt, or go to
**System Settings → Privacy & Security → Files and Folders** and enable
**Downloads Folder** for the program that appears there (here: Python).

To be sure the grant applies to the *launchd* context — not just to your Terminal —
trigger the same access from a throwaway LaunchAgent and check its output:

```xml
<!-- ~/Library/LaunchAgents/com.example.downloads-check.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.example.downloads-check</string>
  <key>ProgramArguments</key><array>
    <string>/bin/bash</string><string>-c</string>
    <string>/path/to/venv/bin/python -c "import pathlib; print(len(list((pathlib.Path.home()/'Downloads').iterdir())), 'entries')"</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>StandardOutPath</key><string>/tmp/downloads-check.log</string>
  <key>StandardErrorPath</key><string>/tmp/downloads-check.log</string>
</dict></plist>
```

```console
$ launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.example.downloads-check.plist
$ cat /tmp/downloads-check.log
5 entries
$ launchctl bootout gui/$(id -u)/com.example.downloads-check
$ rm ~/Library/LaunchAgents/com.example.downloads-check.plist
```

If the log stays empty and the process is alive, the prompt is waiting on screen.

Use the **same interpreter path** your real job uses. A virtualenv's `python` is
usually a symlink to one framework build of Python, and that is the binary macOS
records the permission for.

## Fix it forever: make the script fail fast

A grant can be revoked in System Settings, and the next machine won't
have it. A scheduled job should never hang silently on a permission dialog. Probe the
folder in a background thread and give up after a few seconds:

```python
import os, pathlib, threading

DOWNLOADS = pathlib.Path.home() / "Downloads"

def folder_readable(path: pathlib.Path, timeout: float = 10) -> bool:
    done = threading.Event()

    def peek():
        try:
            next(path.iterdir(), None)
        finally:
            done.set()

    threading.Thread(target=peek, daemon=True).start()
    return done.wait(timeout)

if not folder_readable(DOWNLOADS):
    print(f"Cannot open {DOWNLOADS}: a macOS privacy prompt is probably waiting. "
          "Allow it, or enable it in System Settings → Privacy & Security → "
          "Files and Folders.", flush=True)
    os._exit(3)   # the blocked thread can't be joined, so don't wait for it
```

Two details matter:

- The blocked call can't be cancelled from Python, so the probe runs in a **daemon
  thread** and the main thread only waits on an `Event`.
- Exit with `os._exit()`. It ends the process immediately without running
  interpreter cleanup while another thread is still parked inside a system call —
  there is nothing useful left to clean up anyway.

Now the job ends in ten seconds with a message that says exactly what to click,
instead of holding the morning's work hostage until someone notices.

## Checklist

- A launchd job that touches `~/Downloads`, `~/Desktop` or `~/Documents` needs its
  own privacy grant — the one your Terminal has doesn't count.
- An idle, never-finishing process: run `sample <pid>` before reading code.
- Trigger the access once from a throwaway LaunchAgent to raise and approve the prompt.
- Guard the first access with a timeout so the job fails loudly instead of hanging.

*Tested on macOS 26.5 (Apple silicon), Python 3.12, a LaunchAgent in the `gui/<uid>`
domain.*
