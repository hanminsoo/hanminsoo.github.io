---
title: "launchctl \"Bootstrap failed: 5: Input/output error\": find the real reason without root (and why launchctl load exits 0)"
description: >-
  On macOS, `launchctl bootstrap gui/$(id -u) agent.plist` prints the same
  "Bootstrap failed: 5: Input/output error" for at least six different problems:
  the job is already loaded, the previous copy is still shutting down, the plist
  is broken, it has no Label, the file does not exist, or the label is disabled.
  The real reason is in the unified log, readable without sudo, with
  `/usr/bin/log show` (plain `log` is a zsh builtin). The old `launchctl load`
  and `unload` print "Load failed: 5" but exit 0. `bootout` returns before the
  job has exited, so an immediate re-bootstrap fails too.
tags: [macos, launchd, launchctl, shell]
---

You edit a LaunchAgent, reload it, and get this:

```console
$ T=$(mktemp -d) && cd "$T"
$ cat > com.example.iotest.plist <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.example.iotest</string>
  <key>ProgramArguments</key><array><string>/bin/echo</string><string>hi</string></array>
</dict>
</plist>
EOF
$ launchctl bootstrap gui/$(id -u) com.example.iotest.plist; echo "exit $?"
exit 0
$ launchctl bootstrap gui/$(id -u) com.example.iotest.plist; echo "exit $?"
Bootstrap failed: 5: Input/output error
Try re-running the command as root for richer errors.
exit 5
$ launchctl error 5
5: Input/output error
```

Nothing about the disk is wrong. In every case I tried, error 5 was simply what
`launchctl` printed when launchd refused the request, whatever the reason. You
don't need root to find out why.

## One message, many causes

I ran the same `bootstrap` against six different problems. The terminal output was
identical every time: `Bootstrap failed: 5: Input/output error`, exit 5.
The unified log gave the reason each time. Only the first two share a code:

| What was wrong | What launchd logged |
|---|---|
| Label is already loaded | `failed (37: Operation already in progress)` |
| Previous copy still shutting down after `bootout` | `failed (37: Operation already in progress)` |
| Broken XML (`<true/` missing `>`) | `Could not parse plist ... error = 109: Invalid property list` |
| No `Label` key | `Service plist does not specify a label.` / `110: Invalid or missing service identifier` |
| Path doesn't exist (typo) | `failed (2: No such file or directory)` |
| Label is disabled (`launchctl disable`) | `failed (119: Service is disabled)` |

Domains gave the same message too. `launchctl bootstrap system ...` without root
logged `1: Operation not permitted`. `bootstrap user/$(id -u) ...` with this
plist logged `134: Service cannot load in requested session`.
Both printed the same `Bootstrap failed: 5` in the terminal.

A missing program is **not** on this list. `bootstrap` succeeds (exit 0) and
the failure shows up later, in `launchctl print`:

```console
$ cat > com.example.noprog.plist <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.example.noprog</string>
  <key>ProgramArguments</key><array><string>/no/such/prog</string></array>
  <key>RunAtLoad</key><true/>
</dict>
</plist>
EOF
$ launchctl bootstrap gui/$(id -u) com.example.noprog.plist; echo "exit $?"
exit 0
$ sleep 1; launchctl print gui/$(id -u)/com.example.noprog | grep "last exit"
	last exit code = 78: EX_CONFIG
$ launchctl bootout gui/$(id -u)/com.example.noprog
```

## Read the reason from the log (no sudo)

launchd writes one line per bootstrap request to the unified log, and your own
user can read it:

```console
$ launchctl bootstrap gui/$(id -u) com.example.iotest.plist 2>/dev/null; echo "exit $?"
exit 5
$ /usr/bin/log show --last 1m --style compact --predicate 'process == "launchd" AND eventMessage CONTAINS "Bootstrap by launchctl"' | tail -n 1
2026-11-06 10:01:38.203 Df launchd[1:23b0977] [gui/501 [100023]:] Bootstrap by launchctl[98747] for <private> failed (37: Operation already in progress)
```

(Timestamps, the launchd thread id and the `launchctl[...]` PID will differ on your Mac.)

Two things that tripped me up:

- **Type `/usr/bin/log`, not `log`, in zsh.** `log` is a zsh builtin. `log show ...`
  prints `log: too many arguments` (exit 1) and never runs Apple's tool.
- The label and path show as `<private>` on the "Bootstrap by" line. For parse and
  label errors launchd logs a second line with the path. Drop the
  `CONTAINS` filter to see it:

```console
$ printf '%s\n' '<?xml version="1.0" encoding="UTF-8"?>' '<plist version="1.0"><dict><key>ProgramArguments</key><array><string>/bin/echo</string></array></dict></plist>' > nolabel.plist
$ launchctl bootstrap gui/$(id -u) nolabel.plist 2>/dev/null; echo "exit $?"
exit 5
$ /usr/bin/log show --last 10s --style compact --predicate 'process == "launchd" AND eventMessage CONTAINS "nolabel"'
Timestamp               Ty Process[PID:TID]
2026-11-06 10:01:13.121 E  launchd[1:23b0838] [gui/501:] Could not import service from caller: path = /private/var/folders/xx/xxxxxxxxxxxxxxxxxxxxxxxxxxxx/T/tmp.XXXXXXXXXX/nolabel.plist, caller = launchctl[98658], error = 110: Invalid or missing service identifier
```

(`mktemp -d` makes a random folder under `/var/folders`. I replaced the random parts with `x`/`X`.)

For the plist itself, `plutil -lint` finds the XML errors before launchd sees them:

```console
$ sed 's|<string>hi</string></array>|<string>hi</string></array><key>RunAtLoad</key><true/|' com.example.iotest.plist > bad.plist
$ plutil -lint bad.plist; echo "exit $?"
bad.plist: (Encountered unknown tag true/ on line 6)
exit 1
```

## `launchctl load` prints "failed" and exits 0

The older subcommands report the same error, but their exit status says success:

```console
$ launchctl load com.example.iotest.plist; echo "exit $?"
Load failed: 5: Input/output error
Try running `launchctl bootstrap` as root for richer errors.
exit 0
$ launchctl bootout gui/$(id -u)/com.example.iotest; echo "exit $?"
exit 0
$ launchctl unload com.example.iotest.plist; echo "exit $?"
Unload failed: 5: Input/output error
Try running `launchctl bootout` as root for richer errors.
exit 0
```

So `launchctl load x.plist || echo "failed"` in a script never notices. Use
`bootstrap` and `bootout`, which return non-zero: 5 for a refused bootstrap,
3 (`Boot-out failed: 3: No such process`) for unloading something that isn't loaded.

## `bootout` returns before the job is gone

"Already loaded" is easy to understand. The second 37 is less obvious. Here is a job
that takes a few seconds to stop (it ignores SIGTERM, so launchd waits
`ExitTimeOut` = 5 s and then kills it):

```console
$ cat > com.example.slow.plist <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.example.slow</string>
  <key>ProgramArguments</key><array><string>/bin/sh</string><string>-c</string><string>trap '' TERM; sleep 8</string></array>
  <key>RunAtLoad</key><true/>
  <key>ExitTimeOut</key><integer>5</integer>
</dict>
</plist>
EOF
$ launchctl bootstrap gui/$(id -u) com.example.slow.plist; sleep 1
$ launchctl bootout gui/$(id -u)/com.example.slow; echo "exit $?"
exit 0
$ launchctl bootstrap gui/$(id -u) com.example.slow.plist; echo "exit $?"
Bootstrap failed: 5: Input/output error
Try re-running the command as root for richer errors.
exit 5
$ /usr/bin/log show --last 1m --style compact --predicate 'process == "launchd" AND eventMessage CONTAINS "Bootstrap by launchctl"' | tail -n 1
2026-11-06 10:01:53.640 Df launchd[1:23b0af6] [gui/501 [100023]:] Bootstrap by launchctl[98813] for <private> failed (37: Operation already in progress)
$ sleep 6; launchctl print gui/$(id -u)/com.example.slow; echo "exit $?"
Bad request.
Could not find service "com.example.slow" in domain for user gui: 501
exit 113
```

`bootout` exited 0 straight away, but the service still existed until the
process was gone. A `bootout && bootstrap` reload works for jobs that stop
quickly. It fails for jobs that take a while to stop, like a server finishing
requests or a script that traps TERM. Wait until `launchctl print` stops finding
the label (exit 113, `launchctl error 113` = "Could not find specified
service").

In one of my replays I started the next run while the previous run's slow job
was still stopping. That time the log said `17: File exists` instead of 37,
still with the same `Bootstrap failed: 5` in the terminal. I couldn't make it
happen again on purpose. Waiting for exit 113 avoided it.

## A disabled label

`launchctl disable` is kept by launchd, not in the plist, so editing or
fixing the plist does not undo it. Check and clear it:

```console
$ launchctl disable gui/$(id -u)/com.example.iotest
$ launchctl print-disabled gui/$(id -u) | grep com.example.iotest
		"com.example.iotest" => disabled
$ launchctl bootstrap gui/$(id -u) com.example.iotest.plist 2>/dev/null; echo "exit $?"
exit 5
$ launchctl enable gui/$(id -u)/com.example.iotest
$ launchctl bootstrap gui/$(id -u) com.example.iotest.plist; echo "exit $?"
exit 0
$ launchctl bootout gui/$(id -u)/com.example.iotest
```

After `enable`, the label still shows up in `print-disabled`, now as
`=> enabled`. I found no user-level command that removes the entry, so the
test label stays listed.

## A reload script that says why

This checks the plist, waits for the old copy to go, bootstraps, and on failure
prints launchd's own line for *that* `launchctl` process. It runs
`launchctl` in the background to get its PID, then looks for that PID in the log:

```sh
#!/bin/sh
# agent-reload.sh PLIST - (re)load a LaunchAgent; on failure, print launchd's reason.
set -u
plist=${1:?usage: agent-reload.sh /path/to/agent.plist}
[ -f "$plist" ] || { echo "no such file: $plist" >&2; exit 1; }
plutil -lint "$plist" >/dev/null || exit 1
label=$(plutil -extract Label raw -o - "$plist") || exit 1
service=gui/$(id -u)/$label

if launchctl print "$service" >/dev/null 2>&1; then
  launchctl bootout "$service"
  i=0
  while launchctl print "$service" >/dev/null 2>&1; do   # bootout returns before the job is gone
    i=$((i + 1))
    [ "$i" -le 30 ] || { echo "still unloading after 30 s: $service" >&2; exit 1; }
    sleep 1
  done
fi

launchctl bootstrap "gui/$(id -u)" "$plist" 2>/dev/null &
pid=$!
wait "$pid"
rc=$?
if [ "$rc" -eq 0 ]; then
  echo "loaded $service"
  exit 0
fi
sleep 1   # give the unified log a moment
echo "bootstrap failed with $rc; launchd says:" >&2
/usr/bin/log show --last 1m --style compact \
  --predicate "process == \"launchd\" AND eventMessage CONTAINS \"launchctl[$pid]\"" |
  grep -v '^Timestamp' >&2
exit "$rc"
```

My first version used `if wait "$pid"; then ... fi; rc=$?`. It reported
"failed with 0" and exited 0, because after an `if` with no `else`,
`$?` is 0 even when the condition failed. Save `$?` right after `wait`.

Run against the cases above (saved as `agent-reload.sh` in the same folder):

```console
$ chmod +x agent-reload.sh
$ ./agent-reload.sh com.example.iotest.plist; echo "exit $?"
loaded gui/501/com.example.iotest
exit 0
$ ./agent-reload.sh com.example.iotest.plist; echo "exit $?"
loaded gui/501/com.example.iotest
exit 0
$ ./agent-reload.sh bad.plist; echo "exit $?"
bad.plist: (Encountered unknown tag true/ on line 6)
exit 1
$ ./agent-reload.sh nolabel.plist; echo "exit $?"
nolabel.plist: Could not extract value, error: No value at that key path or invalid key path: Label
exit 1
$ ./agent-reload.sh nothere.plist; echo "exit $?"
no such file: nothere.plist
exit 1
$ launchctl bootstrap gui/$(id -u) com.example.slow.plist; sleep 1
$ ./agent-reload.sh com.example.slow.plist; echo "exit $?"
loaded gui/501/com.example.slow
exit 0
$ launchctl bootout gui/$(id -u)/com.example.iotest; launchctl disable gui/$(id -u)/com.example.iotest
$ ./agent-reload.sh com.example.iotest.plist; echo "exit $?"
bootstrap failed with 5; launchd says:
2026-11-06 10:02:49.920 Df launchd[1:23b0db4] [gui/501 [100023]:] Bootstrap by launchctl[99131] for <private> failed (119: Service is disabled)
exit 5
$ launchctl enable gui/$(id -u)/com.example.iotest
```

The second `com.example.slow` reload took about five seconds. That was the wait
loop, waiting for `ExitTimeOut`.

Clean up the test jobs and the folder (the slow job keeps running about five
more seconds after `bootout`):

```console
$ launchctl bootout gui/$(id -u)/com.example.slow
$ cd / && rm -rf "${T:?}"
```

## Checklist

- `Bootstrap failed: 5: Input/output error` has nothing to do with the disk. It only
  means launchd refused the request. Don't reach for `sudo` first.
- Read the reason: `/usr/bin/log show --last 1m --style compact --predicate
  'process == "launchd" AND eventMessage CONTAINS "Bootstrap by launchctl"'`.
  In zsh, write the full path, because `log` is a builtin.
- `37: Operation already in progress`: already loaded, *or* `bootout` hasn't
  finished. Wait until `launchctl print gui/$(id -u)/LABEL` exits 113.
- `109`: run `plutil -lint`. `110`: add a `Label`. `2`: check the path. `119`:
  `launchctl print-disabled gui/$(id -u)`, then `launchctl enable`.
- A missing program doesn't fail `bootstrap`. Look for `last exit code = 78:
  EX_CONFIG` in `launchctl print`.
- `launchctl load`/`unload` print "failed" and exit 0. Use
  `bootstrap`/`bootout` in scripts.

*Tested on 2026-11-06 on macOS 26.5.2 (build 25F84, Apple silicon), zsh 5.9,
`/bin/sh` (bash 3.2.57), `launchctl` and `plutil` as shipped, user LaunchAgents
in the `gui/501` domain loaded from a temporary folder (not from
`~/Library/LaunchAgents`). Not tested: LaunchDaemons as root, plists in
`~/Library/LaunchAgents` (ownership/permission checks there may give other
errors), other macOS versions.*
