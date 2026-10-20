---
title: "`while pgrep -f` wait loop keeps waiting after the job exits on macOS: the waiters match each other (wait on the PID instead)"
description: >-
  A shell loop that waits for a job with `while pgrep -f job.sh; do sleep 1; done`
  kept waiting after the job had exited. On macOS pgrep skips its own ancestors,
  so a waiter doesn't match itself, but it does match every other process whose
  command line contains the pattern, such as a second waiter or a `tail -f` on
  the script. Two waiters were still waiting for each other a minute after the
  job ended. Fixed by recording the job's PID and waiting with `kill -0` (or `wait`).
tags: [shell, macos, bash, pgrep]
---

You start a long job in the background, and then, in another shell, a
follow-up that should run when it's done:

```bash
bash -c 'while pgrep -f nightly-q7.sh >/dev/null; do sleep 1; done; echo "report starts"'
```

The job finishes, its log says so, and **the follow-up doesn't start**. The
loop is still sleeping, and `pgrep -f` keeps saying the job is running. Below
is a reproduction with a stand-in job, what pgrep was actually matching, and
the fix: wait on the job's PID, not on a name.

## The stand-in job

```bash
#!/bin/bash
# nightly-q7.sh -- stands in for a long job
sleep 3
echo "batch finished"
```

I gave it an unusual name and checked nothing else on the machine matched
before each run (`pgrep -lf nightly-q7` printed nothing and exited 1).
Otherwise any unrelated process with that name in its arguments would keep
the waiters below waiting too.

## One waiter works

With a single waiter, the loop ends as expected:

```console
$ ./nightly-q7.sh > n.log 2>&1 &
$ bash -c 'while pgrep -f nightly-q7.sh >/dev/null; do sleep 1; done; echo "next job starts"' > w.log 2>&1 &
$ sleep 6; cat n.log w.log
batch finished
next job starts
```

You might expect it to hang here: the waiter is `bash -c '... nightly-q7.sh ...'`,
so its own command line contains the pattern. It doesn't, because macOS pgrep
leaves out its ancestors by default. From `man pgrep`:

```
-a   Include process ancestors in the match list.  By
     default, the current pgrep or pkill process and all of
     its ancestors are excluded (unless -v is used).
```

With `-a`, the waiter does match itself (run with no job running; my
output also listed the shell that started `bash -c`, trimmed here):

```console
$ bash -c 'pgrep -lf nightly-q7.sh; echo "default exit $?"; pgrep -alf nightly-q7.sh; echo "-a exit $?"'
default exit 1
76447 bash -c pgrep -lf nightly-q7.sh; echo "default exit $?"; pgrep -alf nightly-q7.sh; echo "-a exit $?"
-a exit 0
```

The ancestor rule only protects a waiter from *itself*. Any process that isn't
its ancestor and has the pattern anywhere in its arguments counts as "the job is
still running".

## Two waiters: each keeps the other waiting

Give the job two follow-ups, each with its own wait loop:

```console
$ ./nightly-q7.sh > n.log 2>&1 &
$ bash -c 'while pgrep -f nightly-q7.sh >/dev/null; do sleep 1; done; echo "report starts"' > w1.log 2>&1 &
$ bash -c 'while pgrep -f nightly-q7.sh >/dev/null; do sleep 1; done; echo "upload starts"' > w2.log 2>&1 &
$ sleep 60; cat n.log w1.log w2.log
batch finished
$ kill -0 77499 2>/dev/null && echo "job alive" || echo "job gone"
job gone
$ pgrep -lf nightly-q7
77500 bash -c while pgrep -f nightly-q7.sh >/dev/null; do sleep 1; done; echo "report starts"
77501 bash -c while pgrep -f nightly-q7.sh >/dev/null; do sleep 1; done; echo "upload starts"
$ ps -o pid,%cpu,time,etime -p 77500,77501
  PID  %CPU      TIME ELAPSED
77500   0.0   0:00.08   01:00
77501   0.0   0:00.08   01:00
```

The job (77499, the `$!` of the first line) finished after 3 seconds. A
minute later neither follow-up had started, and both waiters were still
there, idle. What `pgrep` finds is the two waiters: 77500 sees 77501, and
77501 sees 77500. Neither is the other's ancestor, so neither is excluded.
Neither loop can end until the other one does, so I'd expect them to wait
indefinitely; I stopped them after the minute.

Any other process with the pattern in its arguments does the same, for
example a `tail -f` on the script itself:

```console
$ tail -f nightly-q7.sh >/dev/null &
$ bash -c 'pgrep -lf nightly-q7.sh'
77500 bash -c while pgrep -f nightly-q7.sh >/dev/null; do sleep 1; done; echo "report starts"
77501 bash -c while pgrep -f nightly-q7.sh >/dev/null; do sleep 1; done; echo "upload starts"
77933 tail -f nightly-q7.sh
```

I only tested macOS. I didn't check how pgrep on Linux treats a parent
`bash -c`, so I'm not making a claim about Linux.

To stop the stuck waiters, use the PIDs you got when you started them (`$!`):

```console
$ kill 77933 77500 77501
$ ps -p 77933,77500,77501; echo "exit $?"
  PID TTY           TIME CMD
exit 1
```

Don't kill them by a name pattern. A pattern has the same matching problem
the other way round, and on a machine running other scripts it can stop
things you didn't mean to stop.

## The fix: record the PID, wait on the PID

Start the job through a small wrapper that writes its PID to a file:

```bash
#!/bin/bash
# start-job.sh -- start the job in the background and record its PID
./nightly-q7.sh > nightly-q7.log 2>&1 &
echo $! > nightly-q7.pid
```

and have every follow-up wait on that PID:

```bash
#!/bin/bash
# wait-job.sh -- wait for the PID in the pidfile, not for a name
pid=$(cat nightly-q7.pid)
while kill -0 "$pid" 2>/dev/null; do sleep 1; done
echo "$1 starts"
```

`kill -0` sends no signal; it only checks that the process exists and that you
may signal it. The same two-waiter setup now finishes:

```console
$ ./start-job.sh
$ ./wait-job.sh report > w1.log 2>&1 &
$ ./wait-job.sh upload > w2.log 2>&1 &
$ sleep 6; cat nightly-q7.log w1.log w2.log
batch finished
report starts
upload starts
$ ps -p 77975,77976; echo "exit $?"
  PID TTY           TIME CMD
exit 1
```

(77975 and 77976 are the `$!` of the two waiters; `ps -p` found neither.)

### If it's the same shell: `wait`

When the follow-up runs in the shell that started the job, you don't need a
loop at all:

```console
$ ./nightly-q7.sh > same.log 2>&1 &
$ wait $!; echo "wait exit $?"
wait exit 0
$ cat same.log
batch finished
```

`wait` also gives you the job's exit status. It only works for your own
children; from another shell it fails right away:

```console
$ ./start-job.sh; bash -c 'wait "$(cat nightly-q7.pid)"; echo "wait exit $?"'
bash: wait: pid 76650 is not a child of this shell
wait exit 127
```

That's what the pidfile + `kill -0` loop is for.

### Two caveats of `kill -0`

**A process you can't signal looks finished.** `kill -0` fails with
"Operation not permitted" for a process owned by another user (here PID 1,
launchd, owned by root), and the loop above would treat that as "done".
`ps -p` doesn't have this problem:

```console
$ bash -c 'kill -0 1; echo "kill -0 exit $?"; ps -p 1 >/dev/null; echo "ps -p exit $?"'
bash: line 0: kill: (1) - Operation not permitted
kill -0 exit 1
ps -p exit 0
```

If the job runs as another user (for example a daemon), use
`while ps -p "$pid" >/dev/null; do sleep 1; done`.

**PIDs get reused.** After the job exits, the system can give the same number to
a new process, and a waiter that checks only now and then could keep waiting
on a stranger. I didn't reproduce this; with a 1-second poll and a short
job it is unlikely, but for jobs you wait on for hours, also write a "done"
file at the end of the job and check that.

## Checklist

- `while pgrep -f NAME` waits for **every** process whose arguments contain
  NAME, e.g. another waiter or a `tail -f NAME`.
- On macOS pgrep skips itself and its ancestors (`-a` turns that off), so a
  single waiter works and the bug shows up only when a second matcher appears.
- Two waiters with the same pattern wait for each other (still waiting after
  a minute in my test, at 0.0% CPU).
- Before trusting a pattern, run `pgrep -lf NAME` and read what it matches.
- Fix: record `$!` in a pidfile when starting the job; wait with
  `while kill -0 "$pid" 2>/dev/null; do sleep 1; done`.
- Same shell: `wait "$pid"`, which also returns the job's exit status.
- Job owned by another user: `ps -p "$pid"` instead of `kill -0`.
- Stop stuck loops by the PID you got from `$!`, never by a name pattern.

*Tested on macOS 26.5.2 (Apple silicon) with the system `/usr/bin/pgrep`,
`/bin/bash` 3.2.57. PIDs come from separate runs. Linux was not
tested.*
