---
title: "`timeout: command not found` on macOS: the perl alarm one-liner leaves child processes running (a process-group replacement)"
description: >-
  macOS has no `timeout` command, so `timeout 2 cmd` fails with "command not
  found" (exit 127). The usual replacement, `perl -e 'alarm shift; exec @ARGV'`,
  does stop the command, but only that one process. A script's child process
  kept running after the timeout, re-parented to PID 1. A short perl script
  that runs the command in its own process group and sends TERM to the whole
  group fixes this, exiting 124 like GNU timeout. Tested with the system perl
  5.34 on macOS 26.
tags: [macos, shell, perl, zsh, bash]
---

A script copied from a Linux box, or a command line from an answer online,
stops at the first line:

```console
$ timeout 2 sleep 5; echo "exit $?"
zsh: command not found: timeout
exit 127
```

`timeout` is part of GNU coreutils, and macOS doesn't ship it. There is no
`/usr/bin/timeout` (`ls /usr/bin/timeout` prints `No such file or directory`). The perl one-liner that usually comes up as a
replacement works for a single command, but when the command starts
children of its own, as a script usually does, **the children keep running
after the timeout**. Below: the reproduction, and a short (under 30 lines) perl script that
stops the whole group.

## The one-liner works for a single process

```console
$ perl -e 'alarm shift; exec @ARGV' 2 sleep 5; echo "exit $?"
zsh: alarm      perl -e 'alarm shift; exec @ARGV' 2 sleep 5
exit 142
```

`alarm 2` asks the kernel to send SIGALRM in 2 seconds, and `exec`
replaces perl with `sleep`, so the signal lands on `sleep` and kills it.
The exit status is 128 + 14 (SIGALRM). The `zsh: alarm` line is zsh
reporting how the command died.

## Children survive it

Here is a stand-in for a real script. It starts a child, writes the child's
PID to a file so we can check on it, and waits:

```bash
#!/bin/bash
# job.sh -- stands in for a script that starts a child and waits for it
sleep 30 &
echo $! > child.pid
wait
echo "job finished"
```

Run it under the one-liner with a 2-second limit, then look at the child:

```console
$ perl -e 'alarm shift; exec @ARGV' 2 bash job.sh; echo "exit $?"
zsh: alarm      perl -e 'alarm shift; exec @ARGV' 2 bash job.sh
exit 142
$ ps -o pid,ppid,command -p "$(cat child.pid)"
  PID  PPID COMMAND
76758     1 sleep 30
```

The signal killed `bash job.sh` and nothing else. Its child, `sleep 30`, is
still running, and its parent is now PID 1 (launchd). In a real job, that
leftover could be a download, a build, or a server still holding a port.
(PIDs differ from run to run.) Clean it up by its PID:

```console
$ kill "$(cat child.pid)"
```

(Check the PID with `ps -p "$(cat child.pid)"` first if some time has passed; PIDs get reused.)

## A replacement that stops the process group

Every process belongs to a process group, and `kill` with a negative
number signals the whole group. So: start the command in a new group of
its own, and when time is up, signal the group. Children started by the
command join its group unless they change it themselves.

```perl
#!/usr/bin/perl
# ptimeout SECONDS COMMAND [ARGS...]
# Runs COMMAND in its own process group. If it is still running after
# SECONDS, sends TERM to the whole group and exits 124, like GNU timeout.
use strict; use warnings;
my $secs = shift @ARGV;
unless (defined $secs && @ARGV) {
    print STDERR "usage: ptimeout SECONDS COMMAND [ARGS...]\n";
    exit 125;
}
my $pid = fork() // die "fork: $!\n";
if ($pid == 0) {
    setpgrp(0, 0);
    no warnings 'exec';
    exec { $ARGV[0] } @ARGV;
    print STDERR "ptimeout: $ARGV[0]: $!\n";
    exit 127;
}
setpgrp($pid, $pid);    # also from the parent, so the group exists before the alarm
my $timed_out = 0;
$SIG{ALRM} = sub { $timed_out = 1; kill 'TERM', -$pid };
$SIG{$_} = sub { kill 'TERM', -$pid } for qw(INT TERM HUP);
alarm $secs;
while (waitpid($pid, 0) == -1 && $!{EINTR}) { }
my $status = $?;
alarm 0;
exit 124 if $timed_out;
exit(($status & 127) ? 128 + ($status & 127) : $status >> 8);
```

Save it as `ptimeout` next to `job.sh`, then:

```console
$ chmod +x ptimeout
$ ./ptimeout 2 bash job.sh; echo "exit $?"
exit 124
$ ps -p "$(cat child.pid)" >/dev/null; echo "ps exit $?"
ps exit 1
```

This time the child is gone too (`ps -p` exits 1 when there is no such
process). The other cases:

```console
$ ./ptimeout 5 sleep 1; echo "exit $?"
exit 0
$ ./ptimeout 5 bash -c 'exit 3'; echo "exit $?"
exit 3
$ ./ptimeout 5 no-such-cmd; echo "exit $?"
ptimeout: no-such-cmd: No such file or directory
exit 127
$ ./ptimeout; echo "exit $?"
usage: ptimeout SECONDS COMMAND [ARGS...]
exit 125
```

A command that finishes in time returns right away with its own exit
status. The 124/125/127 codes follow GNU timeout's documented ones, so
scripts that check for 124 keep working. I took those codes from GNU's
documentation and didn't compare against GNU timeout itself, because it
isn't installed here.

Since the command now runs in a different process group, a signal sent to
`ptimeout` would no longer reach it on its own. That's what the INT/TERM/HUP
handlers are for: they pass the signal on to the group. Checked here by
sending INT to `ptimeout` while `job.sh` was running:

```console
$ bash -c './ptimeout 30 bash job.sh & p=$!; sleep 1; kill -INT "$p"; wait "$p"; echo "exit $?"'
exit 143
$ ps -p "$(cat child.pid)" >/dev/null; echo "ps exit $?"
ps exit 1
```

143 is 128 + 15: the job was stopped with TERM, and its child went with it.

### What it doesn't do

- **No KILL after TERM.** A command that ignores TERM keeps running, and
  `ptimeout` keeps waiting. GNU timeout has `-k` for this. I didn't add it.
- **A child that moves to its own process group** (for example with
  `setsid` or a daemonizing server) escapes the group signal. I didn't test
  this case.
- **Interactive programs that read the terminal** run in a background process
  group, and I haven't checked how they behave. Use it for batch jobs.
- Homebrew's `coreutils` provides GNU timeout as `gtimeout`. It isn't
  installed on this Mac, so I haven't tested it.

## Checklist

- `timeout: command not found` (exit 127): macOS doesn't ship GNU
  `timeout`.
- `perl -e 'alarm shift; exec @ARGV' N cmd` stops `cmd` itself (exit 142,
  SIGALRM), but not processes `cmd` started. Those are re-parented to PID 1
  and keep running.
- To check for leftovers, have the job write its children's PIDs, then
  `ps -p PID`. Kill leftovers by PID, not by name.
- To stop a whole job, run it in its own process group (`setpgrp`) and
  signal the group with a negative PID (`kill 'TERM', -$pid`).
- Forward INT/TERM/HUP to the group, or a `kill` on the wrapper won't
  reach the job (I tested with `kill -INT`, not a terminal Ctrl-C).
- Exit 124 on timeout keeps scripts written for GNU timeout working.

*Tested on macOS 26.5.2 (Apple silicon) with the system perl
(`This is perl 5, version 34, subversion 1 (v5.34.1) built for
darwin-thread-multi-2level`), `/bin/zsh` (zsh 5.9 (arm64-apple-darwin25.0))
and `/bin/bash` (GNU bash, version 3.2.57(1)-release (arm64-apple-darwin25)).
GNU coreutils `timeout`/`gtimeout` and Linux were not tested.*
