---
title: "macOS port 5000 \"Address already in use\" (Errno 48) or 403 Forbidden on localhost: ControlCenter (AirTunes) has it, and `netstat -tlnp` doesn't work"
description: >-
  On macOS 26, `python3 -m http.server 5000` failed with "OSError: [Errno 48]
  Address already in use", and `curl -i http://localhost:5000/` returned an
  empty 403 with "Server: AirTunes". ControlCenter listens on *:5000 and
  *:7000 for both IPv4 and IPv6. A server bound to 127.0.0.1:5000 still starts,
  but then `localhost` (::1) reaches AirTunes and gets 403; Node's
  listen(5000, "localhost") binds ::1 and the reverse happens. `netstat -tlnp`
  exits 64 ("option requires an argument -- p") and `ss` doesn't exist; use
  `lsof -nP -iTCP:5000 -sTCP:LISTEN` or `netstat -anv -p tcp`.
tags: [macos, networking, lsof, netstat, python, node]
---

A local dev server on port 5000 won't start on this Mac, and nothing you
started is using the port:

```console
$ python3 -m http.server 5000
Traceback (most recent call last):
...
OSError: [Errno 48] Address already in use
```

Or the server does start (bound to one address, as below), and `curl` gets
an empty **403 Forbidden** that your app never produced:

```console
$ curl -i http://localhost:5000/
HTTP/1.1 403 Forbidden
Content-Length: 0
Server: AirTunes/950.7.1
X-Apple-ProcessingTime: 1
X-Apple-RequestReceivedTimestamp: 1195481073
```

(The two `X-Apple-` values differed between my requests.) `Server: AirTunes` is
the clue. On this Mac, macOS's own **ControlCenter**
process listens on ports 5000 and 7000, on all interfaces, for IPv4 and
IPv6. Below is how I found that without `netstat -tlnp`, why 127.0.0.1 and
`localhost` gave different answers, and what I changed.

## `netstat -tlnp` and `ss` don't work here

The Linux way to ask "who is listening on this port" fails on macOS:

```console
$ netstat -tlnp
-tlnp: option requires an argument -- p
Usage:	netstat [-AaLlnW] [-f address_family | -p protocol]
	netstat [-gilns] [-f address_family]
	netstat -i | -I interface [-w wait] [-abdgRtS]
	netstat -s [-s] [-f address_family | -p protocol] [-w wait]
	netstat -i | -I interface -s [-f address_family | -p protocol]
	netstat -m [-m]
	netstat -r [-Aaln] [-f address_family]
	netstat -rs [-s]
$ echo $?
64
$ ss -tlnp
zsh: command not found: ss
```

On macOS's `netstat`, `-p` takes a protocol name (`-p tcp`), and `-l`
means "print full IPv6 address", not "listening" (`man netstat`). So even
`netstat -tln` runs, but it lists your established connections too, not
just listeners.

`netstat` and `lsof` live in `/usr/sbin`. A normal Terminal has that on
`PATH`; if a script says `lsof: command not found`, call `/usr/sbin/lsof`.

## Find the listener with `lsof`

```console
$ lsof -nP -iTCP:5000 -sTCP:LISTEN
COMMAND   PID USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
ControlCe 670   me   11u  IPv4 0xb734fa7e0696bb17      0t0  TCP *:5000 (LISTEN)
ControlCe 670   me   12u  IPv6 0xf978d71366c0e326      0t0  TCP *:5000 (LISTEN)
```

(PIDs and `DEVICE` values will differ on your Mac.) `ControlCe` is
`lsof` cutting the command name to 9 characters. The full path:

```console
$ ps -o comm= -p "$(lsof -t -iTCP:5000 -sTCP:LISTEN)"
/System/Library/CoreServices/ControlCenter.app/Contents/MacOS/ControlCenter
```

Port 7000 answers the same way:

```console
$ curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:7000/
403
```

The flags matter:

- `-n` and `-P` print numbers. Without `-P`, port 8765 showed up as
  `localhost:ultraseek-http` in my test (a name from `/etc/services`), which
  makes a `grep 8765` miss it.
- `-sTCP:LISTEN` keeps only listening sockets. Without it, `-i :PORT` also
  lists every **client** connected to that port (see the `kill` trap below).

`netstat` can show the owner too, but only with `-v`, in a `process:pid`
column far to the right. Note the port comes after a **dot**, not a colon:

```console
$ netstat -anv -p tcp | awk '$4 ~ /\.5000$/ && /LISTEN/ {print $1, $4, $11}' | sort
tcp4 *.5000 ControlCenter:670
tcp6 *.5000 ControlCenter:670
```

In my setup, a listener owned by **root** showed up in `netstat -anv` but not
in `lsof` run as my user (I did not try `lsof` with
`sudo`). If `lsof` shows nothing and the bind still fails, check
`netstat -anv`.

## Why 127.0.0.1 works and `localhost` gets 403

ControlCenter is bound to the wildcard address (`*:5000`). A socket bound to
a *specific* address can still take the same port if it sets `SO_REUSEADDR`.
This script tries each address with and without it:

```python
# bindtest.py
import socket, sys
host, reuse = sys.argv[1], sys.argv[2] == "reuse"
fam = socket.AF_INET6 if ":" in host else socket.AF_INET
s = socket.socket(fam, socket.SOCK_STREAM)
if reuse:
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
try:
    s.bind((host, 5000))
    print(host, "reuse" if reuse else "no-reuse", "-> bound")
except OSError as e:
    print(host, "reuse" if reuse else "no-reuse", "->", e)
```

```console
$ for h in 127.0.0.1 0.0.0.0 ::1 ::; do for r in reuse no; do python3 bindtest.py $h $r; done; done
127.0.0.1 reuse -> bound
127.0.0.1 no-reuse -> [Errno 48] Address already in use
0.0.0.0 reuse -> [Errno 48] Address already in use
0.0.0.0 no-reuse -> [Errno 48] Address already in use
::1 reuse -> bound
::1 no-reuse -> [Errno 48] Address already in use
:: reuse -> [Errno 48] Address already in use
:: no-reuse -> [Errno 48] Address already in use
```

Python's `http.server` sets `SO_REUSEADDR`, so binding it to `127.0.0.1`
works. But now two processes own port 5000, and which one you reach depends
on the address the client uses:

```console
$ python3 -m http.server 5000 --bind 127.0.0.1 >/dev/null 2>&1 &
$ P=$!; sleep 1
$ curl -s -o /dev/null -w '127.0.0.1 -> %{http_code}\n' http://127.0.0.1:5000/
127.0.0.1 -> 200
$ curl -s -o /dev/null -w 'localhost -> %{http_code} %{remote_ip}\n' http://localhost:5000/
localhost -> 403 ::1
$ kill $P
```

`curl` resolved `localhost` to `::1` first. Nothing of mine listens on
`::1`, so ControlCenter's IPv6 socket answered.

Node does the opposite. In my test, `listen(5000)` with no host failed with
`EADDRINUSE`, and `listen(5000, "localhost")` bound **only** `::1`:

```console
$ node -e 'const s=require("http").createServer((q,r)=>r.end("node\n")); s.listen(5000,"localhost",()=>console.log("bound", s.address()))' &
bound { address: '::1', family: 'IPv6', port: 5000 }
$ P=$!; sleep 1
$ curl -s -o /dev/null -w '127.0.0.1 -> %{http_code}\n' http://127.0.0.1:5000/
127.0.0.1 -> 403
$ curl -s -o /dev/null -w 'localhost -> %{http_code} %{remote_ip}\n' http://localhost:5000/
localhost -> 200 ::1
$ kill $P
```

So the same app works or gets 403 depending on whether you type
`127.0.0.1` or `localhost`. In both cases the 403 has `Content-Length: 0`
and no log line in your server, because the request never reached it.

## The fix: use another port

I moved the dev servers off 5000 (and 7000). Check that the new port is
free first: `lsof` prints nothing and exits 1:

```console
$ lsof -nP -iTCP:8765 -sTCP:LISTEN; echo "exit $?"
exit 1
$ python3 -m http.server 8765 --bind 127.0.0.1 >/dev/null 2>&1 &
$ P=$!; sleep 1
$ curl -s -o /dev/null -w 'localhost -> %{http_code} %{remote_ip}\n' http://localhost:8765/
localhost -> 200 127.0.0.1
$ kill $P
```

With nothing on `::1:8765`, `curl` tried `::1`, got refused, and fell back
to `127.0.0.1`. That fallback is why `localhost` usually "just works", and
why it fails on 5000: there, `::1` *does* answer, with AirTunes.

If you need port 5000, the `Server: AirTunes` header points at AirPlay. The
usual advice is to switch off **AirPlay Receiver** in System Settings. I did
not change that setting on this Mac, so I haven't tested it.

## Don't `kill $(lsof -t -i :PORT)`

`lsof -t` prints only PIDs, which makes `kill "$(lsof -t …)"` tempting. But
without `-sTCP:LISTEN`, `-i :PORT` matches **both ends** of every connection
on that port. With a server on 8765 and one client holding a connection open:

```console
$ python3 -m http.server 8765 --bind 127.0.0.1 >/dev/null 2>&1 &
$ S=$!; sleep 1
$ python3 -c 'import socket,time; s=socket.create_connection(("127.0.0.1",8765)); time.sleep(120)' &
$ C=$!; sleep 1
$ lsof -nP -i :8765
COMMAND   PID USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
Python  18376   me    4u  IPv4 0xaf925289b76679b8      0t0  TCP 127.0.0.1:8765 (LISTEN)
Python  18376   me    5u  IPv4 0x7570a9b1b262b905      0t0  TCP 127.0.0.1:8765->127.0.0.1:64104 (ESTABLISHED)
Python  18424   me    3u  IPv4 0x47fceffcda9c39af      0t0  TCP 127.0.0.1:64104->127.0.0.1:8765 (ESTABLISHED)
$ lsof -t -i :8765
18376
18424
$ lsof -t -iTCP:8765 -sTCP:LISTEN
18376
$ kill $C $S
```

`lsof -t -i :8765` returned the client's PID too. A browser tab with that
page open would be on that list. On port 5000, "the thing on the port" is
ControlCenter itself. Look at the `lsof` output before killing anything, and
kill the PID you started (`$!`), not whatever matches.

## Checklist

- `Errno 48` / `EADDRINUSE` on 5000 or 7000 on macOS, or an empty 403 with
  `Server: AirTunes`: ControlCenter has the port (`*:5000`, IPv4 and IPv6).
- `netstat -tlnp` exits 64 here; `ss` doesn't exist. Use
  `lsof -nP -iTCP:PORT -sTCP:LISTEN`, or `netstat -anv -p tcp` and read the
  `process:pid` column (port after a dot).
- `lsof` without `sudo` didn't show a root-owned listener in my setup;
  `netstat -anv` did.
- Binding `127.0.0.1:5000` (with `SO_REUSEADDR`) succeeds, but then
  `localhost` → `::1` → AirTunes 403. Node's `"localhost"` binds `::1`, so
  `127.0.0.1` gets the 403 instead.
- Fix: use another port, check it with `lsof` first. Open the server at the
  same address it's bound to.
- `lsof -t -i :PORT` includes clients. Add `-sTCP:LISTEN`, read before
  killing, prefer the PID you started.

*Tested on 2026-10-25 on macOS 26.5.2 (build 25F84, Apple silicon) with
`/usr/sbin/lsof` 4.91, `/usr/sbin/netstat`, `/usr/bin/python3` 3.8.2, Node
v24.14.1 and curl 8.7.1, in zsh 5.9 with
`PATH=/usr/bin:/bin:/usr/sbin:/sbin`. AirPlay Receiver was left at its
setting on this Mac (on: ControlCenter was listening); switching it off was not
tested. Linux, Intel Macs and older macOS versions were not tested.*
