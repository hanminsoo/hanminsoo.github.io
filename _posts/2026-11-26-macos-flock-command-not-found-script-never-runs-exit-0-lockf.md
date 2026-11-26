---
title: "macOS `flock: command not found` — the run-once lock that skips every run (exit 0); use `lockf`"
description: >-
  macOS has no flock command. The Linux idiom `exec 9>/tmp/job.lock; flock
  -n 9 || { echo "another run in progress"; exit 0; }` prints "flock:
  command not found", then "another run in progress", and exits 0 — the job
  never runs and nothing looks wrong. `man flock` opens the flock(2) system
  call. macOS ships /usr/bin/lockf: `lockf -t 0 FILE cmd` (exit 75 when
  locked) or `lockf -s -t 0 9` on an open fd inside the script. Killing the
  lockf wrapper releases the lock while the job keeps running; a mkdir lock
  stays after kill -9; a bad lock path also reads as "already running"
  unless you check for 75; background children keep the fd lock until you
  close fd 9.
tags: [macos, flock, lockf, shell, bash, launchd, bsd]
---

A backup script copied from a Linux box, with the usual "only one copy at a
time" guard, run on a Mac:

```console
$ ./job-linux.sh; echo "exit $?"
./job-linux.sh: line 3: flock: command not found
another run in progress, skipping
exit 0
```

Nothing else was running. The script printed its "skipping" line, exited 0,
and the backup never started. Run on a schedule where nobody reads stderr,
the only thing you see is a job that "succeeds" every time and does nothing.

## Reproduce

Everything below runs in a scratch folder; the lock file is `/tmp/job.lock`,
as in the usual Linux examples.

```console
$ T=$(mktemp -d)
$ cd "$T"
```

Save this as `job-linux.sh` (the common Linux `flock` guard):

```bash
#!/bin/bash
exec 9>/tmp/job.lock
flock -n 9 || { echo "another run in progress, skipping"; exit 0; }

echo "backup started $(date +%T)"
sleep 2
echo "backup done"
```

```console
$ chmod +x job-linux.sh
$ ./job-linux.sh; echo "exit $?"
./job-linux.sh: line 3: flock: command not found
another run in progress, skipping
exit 0
```

What happens:

- `flock` is not found, so the shell returns 127 for that command:

  ```console
  $ bash -c 'flock -n 9'; echo "exit $?"
  bash: flock: command not found
  exit 127
  ```

- `||` runs its right side for *any* non-zero status, not just "lock is
  taken". The script cannot tell "busy" from "no such command", prints the
  busy message and exits 0.

There is no `flock` command in the standard system folders (macOS 26.5.2):

```console
$ command -v flock; echo "exit $?"
exit 1
$ ls /bin/flock /sbin/flock /usr/bin/flock /usr/sbin/flock
ls: /bin/flock: No such file or directory
ls: /sbin/flock: No such file or directory
ls: /usr/bin/flock: No such file or directory
ls: /usr/sbin/flock: No such file or directory
```

`man flock` is misleading: on this Mac (Command Line Tools installed) it
opens a page, but it is the C system call, section 2:

```console
$ man -w flock
/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/share/man/man2/flock.2
$ man 1 flock; echo "exit $?"
No manual entry for flock
exit 1
```

## What macOS has instead: `lockf`

`/usr/bin/lockf` (shell_cmds, the FreeBSD tool). Its man page says "BSD-style
locking is used, as described in flock(2); the mere existence of the file is
not considered to constitute a lock." Two forms:

1. a wrapper: `lockf [-t seconds] FILE command ...`
2. on a file descriptor your script already opened: `lockf [-s] [-t seconds] FD`

`-t 0` means "fail at once if locked". Without `-t`, `lockf` waits for the
lock — per the man page "indefinitely"; the wait is shown below.

### The wrapper form

Save this as `backup.sh`:

```bash
#!/bin/bash
echo "backup started $(date +%T)"
sleep 2
echo "backup done"
```

```console
$ chmod +x backup.sh
$ lockf -t 0 /tmp/job.lock ./backup.sh & first=$!
$ sleep 0.5
backup started 10:13:01
$ lockf -t 0 /tmp/job.lock ./backup.sh; echo "second exit $?"
lockf: /tmp/job.lock: already locked
second exit 75
$ lockf -s -t 0 /tmp/job.lock ./backup.sh; echo "with -s exit $?"
with -s exit 75
$ wait $first; echo "first exit $?"
backup done
first exit 0
$ ls /tmp/job.lock
ls: /tmp/job.lock: No such file or directory
```

- A taken lock is exit **75** (`EX_TEMPFAIL`) with "already locked" on
  stderr. `-s` removes the message, not the 75 (the `with -s` line).
- Otherwise `lockf` returns the command's own status:

  ```console
  $ lockf -t 0 /tmp/job.lock sh -c 'exit 3'; echo "exit $?"
  exit 3
  ```

- Without `-k`, `lockf` deletes the lock file when the command ends (the
  `ls` above). The man page recommends `-k` when several processes compete
  for the lock, for lock ordering.
- Without `-t`, the second `lockf` waits and then runs. Its "backup started"
  comes after the first run's "backup done":

  ```console
  $ lockf -t 0 /tmp/job.lock ./backup.sh & first=$!
  $ sleep 0.5
  backup started 10:13:04
  $ lockf /tmp/job.lock ./backup.sh; echo "second exit $?"
  backup done
  backup started 10:13:06
  backup done
  second exit 0
  $ wait $first
  ```

### Trap: kill the wrapper and the lock is gone, the job is not

If something stops the job by the PID it got when starting it, that PID is
`lockf`, not the script. The script survives, re-parented to PID 1, and the
lock is released:

```console
$ lockf -t 0 /tmp/job.lock ./backup.sh & first=$!
$ sleep 0.5
backup started 10:13:08
$ child=$(pgrep -P $first); kill $first; wait $first; echo "lockf exit $?"
lockf exit 143
$ ps -o pid,ppid,command -p $child
  PID  PPID COMMAND
37481     1 /bin/bash ./backup.sh
$ lockf -t 0 /tmp/job.lock ./backup.sh; echo "exit $?"
backup started 10:13:09
backup done
backup done
exit 0
$ sleep 2
```

The new run got the lock while the old `backup.sh` was still going; the
first of the two "backup done" lines is the orphaned old run finishing.
Two backups ran at the same time — exactly what the lock was for.
`kill -9` on the `lockf` PID does the same:

```console
$ lockf -t 0 /tmp/job.lock ./backup.sh & first=$!
$ sleep 0.5
backup started 10:13:13
$ child=$(pgrep -P $first); kill -9 $first; wait $first; echo "lockf exit $?"
lockf exit 137
$ ps -o pid,ppid,command -p $child
  PID  PPID COMMAND
37502     1 /bin/bash ./backup.sh
$ lockf -t 0 /tmp/job.lock ./backup.sh; echo "exit $?"
backup started 10:13:14
backup done
backup done
exit 0
$ sleep 2
```

(Ctrl-C in a terminal was not tested.)

## The fix: lock inside the script, and check for 75

The fd form is the direct replacement for `exec 9>file; flock -n 9`. The
lock belongs to the open file behind fd 9, so it lives as long as
the script (and anything that inherited fd 9). Save this as `job.sh`:

```bash
#!/bin/bash
# Run at most one copy at a time (macOS: lockf instead of flock).
LOCK=/tmp/job.lock

exec 9>"$LOCK" || exit 1
lockf -s -t 0 9
rc=$?
if [ "$rc" -eq 75 ]; then
  echo "another run in progress, skipping"
  exit 0
elif [ "$rc" -ne 0 ]; then
  echo "could not lock $LOCK (lockf exit $rc)" >&2
  exit 1
fi

echo "backup started $(date +%T)"
sleep 2
echo "backup done"
```

```console
$ chmod +x job.sh
$ ./job.sh & first=$!
$ sleep 0.5
backup started 10:13:18
$ ./job.sh; echo "second exit $?"
another run in progress, skipping
second exit 0
$ wait $first; echo "first exit $?"
backup done
first exit 0
$ ls -l /tmp/job.lock
-rw-r--r--  1 me  wheel  0 Nov 26 10:13 /tmp/job.lock
$ ./job.sh; echo "exit $?"
backup started 10:13:20
backup done
exit 0
```

The lock file stays (the man page: "The -k option is implied when a file
descriptor is in use"), and that is fine: the next
run takes the lock normally. Only an open, locked file blocks a run.

Kill the script itself with `kill -9`:

```console
$ ./job.sh & first=$!
$ sleep 0.5
backup started 10:13:22
$ kill -9 $first; wait $first; echo "job exit $?"
job exit 137
$ ./job.sh; echo "exit $?"
another run in progress, skipping
exit 0
$ lsof /tmp/job.lock
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF     NODE NAME
sleep   37553   me    9w   REG   1,16        0 21078426 /private/tmp/job.lock
$ sleep 2
$ ./job.sh; echo "exit $?"
backup started 10:13:26
backup done
exit 0
```

Right after the kill the lock is still held — by the script's `sleep`, which
outlived its parent and inherited fd 9 (see the last section). Once that
process ended, the next run went ahead. No stale lock is left to delete.

### Why check for 75 and not just `||`

`lockf -s -t 0 9 || { echo skipping; exit 0; }` has the same blind spot as
the `flock` line: every failure reads as "busy". A lock path in a folder
that does not exist, for example. Save this as `job-naive.sh`:

```bash
#!/bin/bash
exec 9>/tmp/no-such-dir/job.lock
lockf -s -t 0 9 || { echo "another run in progress, skipping"; exit 0; }
echo "backup done"
```

```console
$ bash job-naive.sh; echo "exit $?"
job-naive.sh: line 2: /tmp/no-such-dir/job.lock: No such file or directory
lockf: cannot lock fd 9: Bad file descriptor
another run in progress, skipping
exit 0
$ lockf -s -t 0 9; echo "exit $?"
lockf: cannot lock fd 9: Bad file descriptor
exit 73
```

`exec 9>` failed, fd 9 was never opened, and `lockf` on a closed fd exits
73 (the last command, run in a shell with no fd 9; `-s` does not hide that
message). The script still said "skipping" with exit 0.
`job.sh` with the same bad path stops at the `exec` line with exit 1:

```console
$ sed 's|^LOCK=.*|LOCK=/tmp/no-such-dir/job.lock|' job.sh > job-bad.sh
$ bash job-bad.sh; echo "exit $?"
job-bad.sh: line 5: /tmp/no-such-dir/job.lock: No such file or directory
exit 1
$ sh job-bad.sh; echo "exit $?"
job-bad.sh: line 5: /tmp/no-such-dir/job.lock: No such file or directory
exit 1
```

Without the `|| exit 1`, bash carries on after a failed `exec` redirection;
`/bin/sh` (bash 3.2.57 in sh mode here) stops by itself:

```console
$ bash -c 'exec 9>/tmp/no-such-dir/job.lock; echo "still running"'; echo "exit $?"
bash: /tmp/no-such-dir/job.lock: No such file or directory
still running
exit 0
$ sh -c 'exec 9>/tmp/no-such-dir/job.lock; echo "still running"'; echo "exit $?"
sh: /tmp/no-such-dir/job.lock: No such file or directory
exit 1
```

## Why not `mkdir /tmp/job.lock.d`?

The other common replacement is a lock directory: `mkdir` is atomic, and a
`trap` removes the directory on exit. Save this as `job-mkdir.sh`:

```bash
#!/bin/bash
mkdir /tmp/job.lock.d 2>/dev/null || { echo "another run in progress, skipping"; exit 0; }
trap 'rmdir /tmp/job.lock.d' EXIT

echo "backup started $(date +%T)"
sleep 2
echo "backup done"
```

```console
$ chmod +x job-mkdir.sh
$ ./job-mkdir.sh & first=$!
$ sleep 0.5
backup started 10:13:28
$ kill $first; wait $first; echo "TERM: exit $?"
TERM: exit 143
$ ls -d /tmp/job.lock.d
ls: /tmp/job.lock.d: No such file or directory
$ ./job-mkdir.sh & first=$!
$ sleep 0.5
backup started 10:13:28
$ kill -9 $first; wait $first; echo "KILL: exit $?"
KILL: exit 137
$ ls -d /tmp/job.lock.d
/tmp/job.lock.d
$ sleep 2
$ ./job-mkdir.sh; echo "exit $?"
another run in progress, skipping
exit 0
$ rmdir /tmp/job.lock.d
```

With SIGTERM bash ran the `EXIT` trap and the directory went away. With
`kill -9` nothing runs the trap: the directory stays, and every
later run skips — again with exit 0 — until someone deletes it by hand. A
`flock(2)` lock goes away when the last process that has the locked file
open closes it or dies (that is what the `lsof` example above showed), so
`lockf` leaves no stale state like this.

## Background children keep the lock

Anything the script starts inherits fd 9, and with it the lock. A helper
left running in the background keeps "another run in progress" true after
the script has exited. Save this as `job-helper.sh`:

```bash
#!/bin/bash
exec 9>/tmp/job.lock
lockf -s -t 0 9 || { echo "another run in progress, skipping"; exit 0; }

sleep 30 >/dev/null 2>&1 &
echo $! > helper.pid
echo "backup done"
```

```console
$ chmod +x job-helper.sh
$ ./job-helper.sh; echo "exit $?"
backup done
exit 0
$ ./job-helper.sh; echo "exit $?"
another run in progress, skipping
exit 0
$ kill "$(cat helper.pid)"
```

Close fd 9 for that one command with `9>&-`, then run it twice again, this
time leaving the first helper alive:

```console
$ sed -i '' 's|^sleep 30 >/dev/null 2>&1 &$|sleep 30 >/dev/null 2>\&1 9>\&- \&|' job-helper.sh
$ grep '^sleep' job-helper.sh
sleep 30 >/dev/null 2>&1 9>&- &
$ ./job-helper.sh; echo "exit $?"
backup done
exit 0
$ h1=$(cat helper.pid)
$ ps -o pid,ppid,command -p "$h1"
  PID  PPID COMMAND
37628     1 sleep 30
$ ./job-helper.sh; echo "exit $?"
backup done
exit 0
$ kill "$h1" "$(cat helper.pid)"
```

The first helper was still running (PPID 1) when the second run started,
and this time the second run went ahead.

I tested this with `lockf` only, not with util-linux `flock`.
(`job-helper.sh` keeps the short `||` check to
stay small; use the `rc` check from `job.sh` in real scripts.)

`lsof /tmp/job.lock` shows who holds the lock, as in the `kill -9` example
above. Stop a holder by its PID, not by a name pattern.

Clean up:

```console
$ rm -f /tmp/job.lock
$ cd /
$ rm -r "${T:?}"
```

## Checklist

- macOS has no `flock` command (`command -v flock` fails; `man flock` is the
  system call). Calling it is "command not found", exit 127.
- `flock -n 9 || { echo busy; exit 0; }` treats "command not found" as
  "busy": the job is skipped every time and exits 0.
- Use `/usr/bin/lockf`. Inside a script: `exec 9>"$LOCK" || exit 1`, then
  `lockf -s -t 0 9`. `-t 0` = don't wait; no `-t` = wait until the lock is
  free ("indefinitely", per the man page).
- Treat only exit 75 as "already running". Anything else (73 for a bad fd,
  1, ...) is an error: log it and exit non-zero.
- The wrapper form `lockf -t 0 FILE cmd` is fine for a one-liner, but
  killing the `lockf` PID leaves the command running without the lock.
- A `mkdir` lock survives `kill -9`; an `flock(2)` lock is released once the
  last process holding the locked file open exits (children that inherited
  fd 9 keep it). The lock
  file existing means nothing; an open, locked file does.
- Background helpers inherit the lock: while one is alive, the next run is
  skipped. Start them with `9>&-` and the next run goes ahead even with the
  helper still running.

Related: [`while pgrep -f job.sh` wait loop never ends](/2026/10/pgrep-f-wait-loop-never-ends-wait-on-pid/)
and [macOS `timeout: command not found`](/2026/10/macos-timeout-command-not-found-perl-alarm-child-keeps-running/).

*Tested on 2026-11-26 on macOS 26.5.2 (Apple silicon, APFS) with
`/usr/bin/lockf` (shell_cmds-329), `/bin/bash` 3.2.57 (also as `/bin/sh`),
`/usr/sbin/lsof`, zsh 5.9, in `env -i` with
`PATH=/usr/bin:/bin:/usr/sbin:/sbin`.
Not tested: util-linux `flock` (Linux or Homebrew), launchd/cron runs of
these scripts, network or non-APFS file systems, `lockf -w`, older macOS
versions.*
