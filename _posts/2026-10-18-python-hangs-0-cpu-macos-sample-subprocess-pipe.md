---
title: "Python script hangs at 0% CPU on macOS: find where with `sample` (and why spindump says it must be run as root)"
description: >-
  A Python script that runs a subprocess stopped doing anything: no CPU, no
  output, no error. macOS's built-in `sample` showed the parent stuck in
  __wait4 and the child stuck in write — the classic subprocess pipe deadlock
  once the child prints more than 64 KiB. spindump refused without root
  ("spindump must be run as root when sampling the live system"). Fixed with
  communicate(); faulthandler gives the Python line.
tags: [python, macos, debugging, subprocess]
---

A Python script **hangs forever at 0% CPU**: no output, no traceback, nothing
in the log. `ps` shows it alive and idle. You want to know *where* it's
waiting without adding prints and restarting it.

On macOS you don't need to install anything for the first look: `/usr/bin/sample`
attaches to a running process for a few seconds and prints where every thread
is. Below I use it on a hang I reproduced on my Mac — a subprocess pipe
deadlock — and show what it can and can't tell you, plus what happens when you
reach for `spindump` instead.

## The hang

The script starts a child process, captures its stdout through a pipe, and
waits for it to finish before reading:

```python
# stuck.py -- waits for a child that is blocked writing to a full pipe
import subprocess, sys
p = subprocess.Popen(
    [sys.executable, "-c", "print('x' * 200_000)"],
    stdout=subprocess.PIPE,
)
p.wait()          # never returns: nobody reads the pipe
print(len(p.stdout.read()))
```

Run it in the background so you get its PID, then look at it:

```console
$ python3 stuck.py &
$ ps -o pid,ppid,%cpu,time -p 52520
  PID  PPID  %CPU      TIME
52520 52518   0.0   0:00.03
$ pgrep -P 52520
52522
```

Alive, 0% CPU, 30 ms of CPU time in total, and it has one child (`52522`).
Nothing is spinning; something is waiting.

## Step 1: `sample` the stuck process

`sample <pid> <seconds>` takes a stack snapshot of every thread about once per
millisecond. Use `-file` so the report lands where you want it (without
`-file`, it printed the report to the terminal *and* saved a copy as
`/tmp/Python_<date>_<id>.sample.txt` on my Mac):

```console
$ sample 52520 1 -file parent.txt
Sampling process 52520 for 1 second with 1 millisecond of run time between samples
Sampling completed, processing symbols...
Sample analysis of process 52520 written to file parent.txt
$ grep -A1 "Sort by top of stack" parent.txt
Sort by top of stack, same collapsed (when >= 5):
        __wait4  (in libsystem_kernel.dylib)        775
```

The "Sort by top of stack" section at the bottom of the report is the fastest
thing to read: it counts which function was on top of the stack in each
sample. Here **all 775 samples** were in `__wait4` — the process is waiting
for a child to exit. The call graph above it shows how it got there (address
columns trimmed):

```console
                                                    775 os_waitpid  (in Python3) + 92
                                                      775 __wait4  (in libsystem_kernel.dylib) + 8
```

`os_waitpid` is the C function behind `os.waitpid()`, which `Popen.wait()`
calls. So: the parent is in `wait()`, waiting for the child. Now sample the
child:

```console
$ sample 52522 1 -file child.txt
Sampling process 52522 for 1 second with 1 millisecond of run time between samples
Sampling completed, processing symbols...
Sample analysis of process 52522 written to file child.txt
$ grep -A1 "Sort by top of stack" child.txt
Sort by top of stack, same collapsed (when >= 5):
        write  (in libsystem_kernel.dylib)        778
```

The child is stuck in `write` (its call graph runs from `builtin_print` down
through `_io_FileIO_write` to `write`). It is trying to print, and the write
doesn't complete.

That's the whole deadlock: **the child waits for someone to read the pipe, and
the parent waits for the child to exit.** A pipe has a fixed buffer; once the
child has filled it, its next `write()` blocks until the other end reads.
`stuck.py` doesn't read until after `wait()` returns, which it never does.

## How big is the pipe?

On my Mac the child finished as long as it wrote at most 65,536 bytes, and hung
at 65,537. This script checks a few sizes with a timeout so it can't hang
itself:

```python
# pipesize.py -- does the child finish if nobody reads its stdout?
import subprocess, sys
for n in (16_384, 65_535, 65_536, 65_537, 200_000):
    p = subprocess.Popen(
        [sys.executable, "-c", f"import sys; sys.stdout.write('x' * {n})"],
        stdout=subprocess.PIPE,
    )
    try:
        p.wait(timeout=3)
        print(n, "finished")
    except subprocess.TimeoutExpired:
        print(n, "stuck")
        p.kill()
    p.communicate()
```

```console
$ python3 pipesize.py
16384 finished
65535 finished
65536 finished
65537 stuck
200000 stuck
```

Same result with Python 3.12. This is why the bug hides in testing: with small
output, `wait()` then `read()` works fine; it only hangs when the child's output
grows past 64 KiB.

## Why not spindump?

`spindump` is the other built-in tool people suggest for hangs (it samples user
*and* kernel stacks). Run as a normal user, it stopped immediately:

```console
$ /usr/sbin/spindump 52520 1 -file spin.txt
spindump must be run as root when sampling the live system
$ echo $?
77
```

(It lives in `/usr/sbin`; a normal login shell has that on `PATH`, so plain
`spindump` gives the same message.) I didn't run it with `sudo`, so I
can't show its output here. For a process you own, `sample` needs no root and
was enough to find this hang.

## What `sample` can't tell you: the Python line

`sample` shows C frames: `_PyEval_EvalFrameDefault`, `call_function`,
`os_waitpid`. It doesn't say which line of *your* script is waiting. For that,
let Python print its own stack on a signal with `faulthandler` — two lines at the
top of the script:

```python
# stuck_fh.py -- same hang, but `kill -USR1 <pid>` prints the Python stack
import faulthandler, signal, subprocess, sys
faulthandler.register(signal.SIGUSR1)

p = subprocess.Popen(
    [sys.executable, "-c", "print('x' * 200_000)"],
    stdout=subprocess.PIPE,
)
p.wait()          # never returns: nobody reads the pipe
print(len(p.stdout.read()))
```

```console
$ python3 stuck_fh.py 2>fh.err &
$ kill -USR1 52343
$ cat fh.err
Current thread 0x00000001f399dd80 (most recent call first):
  File "/Library/Developer/CommandLineTools/Library/Frameworks/Python3.framework/Versions/3.8/lib/python3.8/subprocess.py", line 1762 in _try_wait
  File "/Library/Developer/CommandLineTools/Library/Frameworks/Python3.framework/Versions/3.8/lib/python3.8/subprocess.py", line 1804 in _wait
  File "/Library/Developer/CommandLineTools/Library/Frameworks/Python3.framework/Versions/3.8/lib/python3.8/subprocess.py", line 1079 in wait
  File "stuck_fh.py", line 9 in <module>
```

Line 9 is `p.wait()`. The process keeps running after the dump (SIGUSR1 doesn't
stop it), so you can do this to a long-running service. It only works if the
handler was registered *before* the hang, so it's worth adding to scripts that
run unattended.

## Stopping it

Kill the parent by the PID you got when you started it:

```console
$ kill 52520
Traceback (most recent call last):
  File "<string>", line 1, in <module>
BrokenPipeError: [Errno 32] Broken pipe
```

The traceback is from the child: when the parent died, the read end of the pipe
closed, the child's blocked `write` failed, and the child exited too (`ps -p`
found neither PID afterwards). Kill by PID rather than by process name: a
name pattern also matches every other Python process on the machine, including
services you didn't mean to stop.

## The fix: read while you wait

Use `communicate()`, which reads stdout (and stderr) while waiting for the
child to exit:

```python
# fixed.py -- communicate() reads the pipe while waiting
import subprocess, sys
p = subprocess.Popen(
    [sys.executable, "-c", "print('x' * 200_000)"],
    stdout=subprocess.PIPE,
)
out, _ = p.communicate(timeout=30)
print(len(out), "exit", p.returncode)
```

```console
$ python3 fixed.py
200001 exit 0
```

`subprocess.run(..., capture_output=True)` uses `communicate()` internally and
behaves the same way:

```console
$ python3 -c "import subprocess,sys; r=subprocess.run([sys.executable,'-c',\"print('x' * 200_000)\"], capture_output=True, timeout=30); print(len(r.stdout), 'exit', r.returncode)"
200001 exit 0
```

The `timeout=` doesn't fix the deadlock — `communicate()` does — but it turns
any future hang (a child that really never exits) into an exception instead
of a silent 0% CPU process.

## Checklist

- A process hangs at 0% CPU: `sample <pid> 1 -file out.txt`, then read
  "Sort by top of stack" at the bottom.
- `__wait4` on top means it's waiting for a child process. `pgrep -P <pid>`
  and sample the child too.
- Child in `write` + parent in `wait4` = pipe deadlock. Replace
  `Popen(..., stdout=PIPE)` + `wait()` with `communicate()` or
  `subprocess.run(capture_output=True)`.
- It only shows up once the output passes the pipe buffer (64 KiB on my Mac),
  so small test runs pass.
- `spindump` needs root on a live system; `sample` doesn't for your own
  processes.
- To get the Python line, register `faulthandler` on `SIGUSR1` and
  `kill -USR1 <pid>`.
- Stop the hung process by its PID, not by a name pattern.

*Tested on macOS 26.5.2 (Apple silicon) with Python 3.8.2 (`/usr/bin/python3`,
Command Line Tools); the pipe-size check and the fix were also run on Python
3.12.13 (Homebrew). `sample` and `spindump` are the ones shipped with that macOS.
spindump was not run as root. PIDs and the faulthandler output are from
separate runs, so the PIDs differ.*
