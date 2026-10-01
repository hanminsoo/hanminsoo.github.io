---
title: "Four launchd traps: Desktop scripts, logged-out agents, keychain-less daemons, and PATH"
description: >-
  Things that silently broke my scheduled jobs on macOS: a script in ~/Desktop
  exiting 126, a LaunchAgent that vanished when the login session ended, a
  LaunchDaemon that couldn't read the login keychain, and launchd's minimal PATH.
  How to see each one and what to do instead.
tags: [macos, launchd, keychain]
---

`launchd` is a great scheduler and a terrible narrator. When a job fails it usually
says nothing at all. These are the four failures that cost me the most time on a
Mac mini that runs a small server and a few daily jobs.

The one command to remember for all of them:

```console
$ launchctl print gui/$(id -u)/com.example.job      # a LaunchAgent
$ sudo launchctl print system/com.example.job       # a LaunchDaemon
```

Look at `state` and `last exit code`.

## 1. The script lives in ~/Desktop (or Documents, or Downloads)

I reproduced this one just now with a two-line script:

```console
$ cat ~/Desktop/launchd-desktop-test.sh
#!/bin/bash
echo ran-ok

$ launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.example.desktop-test.plist
$ launchctl print gui/$(id -u)/com.example.desktop-test | grep "last exit"
	last exit code = 126
$ cat /tmp/desktop-test.log
/bin/bash: /Users/me/Desktop/launchd-desktop-test.sh: Operation not permitted
```

The same script runs fine from Terminal. `~/Desktop`, `~/Documents` and
`~/Downloads` are privacy-protected folders; Terminal has been granted access to
them, a job started by launchd has not. If you didn't set `StandardErrorPath`, the
only trace is that exit code.

Interestingly, a job whose `WorkingDirectory` is on the Desktop but whose program
lives elsewhere ran fine in my test — it's reading the script file that fails.

**Fix:** keep anything launchd runs outside the protected folders — a project
directory under your home, `~/bin`, `/usr/local/bin`. (A *different* symptom of
the same protection, a job hanging while it opens `~/Downloads`, is in
[the previous post]({% post_url 2026-10-01-launchd-python-hangs-opening-downloads %}).)

## 2. The LaunchAgent disappeared when nobody was logged in

A LaunchAgent in `~/Library/LaunchAgents` is loaded into your **login session**
(`gui/<uid>`). It runs while that session exists. If the session ends — you log
out, or the GUI session crashes and the Mac sits at the login window — every agent
in it stops, and nothing is written to your job's error log, because the job
didn't crash. It just isn't there.

From across the room a Mac at the login window looks exactly like a Mac that is
working. I lost a home server for two days this way before noticing.

**Fix:** a service that must survive logouts belongs in `/Library/LaunchDaemons`.
You can still run it as your normal user — that's what `UserName` is for — so files
it creates keep the right owner:

```xml
<key>UserName</key><string>me</string>
<key>EnvironmentVariables</key>
<dict>
  <key>HOME</key><string>/Users/me</string>
  <key>PATH</key><string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
</dict>
```

Daemons start with an almost empty environment, so set `HOME` explicitly — many
tools look for their config under `$HOME` and fail in confusing ways without it.

## 3. The daemon can't use the login keychain

Moving to a daemon fixed uptime and broke one feature: the server shells out to a
command-line tool that keeps its login token in the **login keychain**. From the
daemon, the tool reported:

```
Not logged in · Please run /login
```

…and exited 1, while the server itself kept running happily. Setting `HOME` and
`USER` did not help.

The reason: a system daemon runs **outside any login session**. You can see it:

```console
$ id -A        # from Terminal
auid=501
$ id -A        # from the daemon
auid=-1
```

The login keychain belongs to the login session, so it isn't in the daemon's
keychain search list, no matter which user the daemon runs as.

**Fix (without giving up the daemon):** when that one tool is needed, hand the
work *into* the user's session. A daemon running as the same uid is allowed to
bootstrap a short-lived job into `gui/<uid>`; there the keychain opens normally.

```bash
#!/usr/bin/env bash
# run-in-session.sh CMD ARGS... — run CMD inside the logged-in user's session.
set -euo pipefail
if [ "$(id -A | head -1)" != "auid=-1" ]; then exec "$@"; fi   # already in a session

LABEL=com.example.in-session
ROOM="$(mktemp -d)"; trap 'rm -rf "$ROOM"' EXIT
cat > "$ROOM/stdin"
{ printf '#!/bin/bash\n'; printf '%q ' "$@"
  printf '< %q > %q 2> %q\necho $? > %q\n' \
    "$ROOM/stdin" "$ROOM/out" "$ROOM/err" "$ROOM/code"; } > "$ROOM/run"
chmod +x "$ROOM/run"
cat > "$ROOM/job.plist" <<PLIST
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>$LABEL</string>
  <key>ProgramArguments</key><array><string>$ROOM/run</string></array>
  <key>RunAtLoad</key><true/>
</dict></plist>
PLIST
GUI="gui/$(id -u)"
launchctl bootout "$GUI/$LABEL" 2>/dev/null || true
launchctl bootstrap "$GUI" "$ROOM/job.plist"
while [ ! -s "$ROOM/code" ]; do
  launchctl print "$GUI/$LABEL" >/dev/null 2>&1 || { sleep 0.3; [ -s "$ROOM/code" ] || break; }
  sleep 0.2
done
launchctl bootout "$GUI/$LABEL" 2>/dev/null || true
cat "$ROOM/out" 2>/dev/null || true
cat "$ROOM/err" >&2 2>/dev/null || true
exit "$(cat "$ROOM/code" 2>/dev/null || echo 70)"
```

The caller sees the same contract as the original tool: arguments in, stdin in,
stdout/stderr/exit code out. If nobody is logged in, only this one feature fails —
the rest of the daemon stays up, which is the trade I wanted.

Two notes:

- **Read stdout too.** The tool printed "Not logged in" on *stdout* and nothing on
  stderr. If your server only logs stderr, you'll just see "exited with 1".
- Don't "fix" this by turning the daemon back into an agent. That trades a broken
  button for trap #2.

## 4. launchd's PATH is not your PATH

Jobs get a minimal `PATH` (`/usr/bin:/bin:/usr/sbin:/sbin`). Homebrew's
`/opt/homebrew/bin` is not in it, so `ffmpeg`, `node` or `python3` from Homebrew
"aren't installed" as far as your job is concerned.

**Fix:** use absolute paths in `ProgramArguments`, or set `PATH` in
`EnvironmentVariables` (as above). For tools whose location you discover at
install time, write the resolved path into the plist when you install the job,
rather than hoping it's on `PATH` at run time.

## Checklist

| Symptom | Likely trap |
|---|---|
| `last exit code = 126`, "Operation not permitted" | script in a privacy-protected folder |
| Job simply isn't running, no errors anywhere | LaunchAgent and the login session ended |
| A CLI says "not logged in" only under launchd | daemon outside the session, no login keychain |
| "command not found" only under launchd | minimal `PATH` |

*Tested on macOS 26.5. Trap 1 reproduced for this post; traps 2–4 are from running
a home server on the same machine.*
