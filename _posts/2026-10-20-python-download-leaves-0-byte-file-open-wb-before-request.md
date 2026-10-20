---
title: "Python download script leaves a 0-byte file after a 404 or a dropped connection, and the next run skips it: open(path, \"wb\") before the request"
description: >-
  A urllib download that opens the output file before the request leaves an
  empty file when the request fails (HTTP Error 404, Connection refused,
  IncompleteRead), and an "if os.path.exists(path): skip" check then treats it
  as done forever. Streaming with shutil.copyfileobj is worse: a dropped
  connection leaves a truncated file and exits 0. Reproduced against a local
  test server, with a fix: connect first, write to a temp file, check
  Content-Length, then os.replace.
tags: [python, macos, http, filesystems]
---

A download script fails once (a 404, the server was down, the connection
dropped) and leaves behind a **0-byte file** with the right name. On the next
run the script prints "already downloaded" and moves on, so the empty file
stays there until something downstream chokes on it. The cause is the order of
two lines: `open(path, "wb")` runs before the request, and `"wb"` creates (or
empties) the file right away, whether or not any bytes ever arrive.

Below I reproduce it against a small local test server, show a second, quieter
version where a streaming copy writes a **truncated file and exits 0**, and
then the fix I use.

## A test server that fails on demand

Three routes: `/ok` sends 200,000 bytes, `/cut` promises 200,000 bytes in
`Content-Length` but hangs up after 1,000, anything else is a 404.

```python
# server.py — test server: /ok sends 200 KB, /cut promises 200 KB and hangs
# up after 1 KB, anything else is 404.
import sys
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

BODY = b"x" * 200_000

class H(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/ok":
            self.send_response(200)
            self.send_header("Content-Length", str(len(BODY)))
            self.end_headers()
            self.wfile.write(BODY)
        elif self.path == "/cut":
            self.send_response(200)
            self.send_header("Content-Length", str(len(BODY)))
            self.end_headers()
            self.wfile.write(BODY[:1000])
            self.wfile.flush()
            self.close_connection = True
        else:
            self.send_error(404)
    def log_message(self, *a):
        pass

ThreadingHTTPServer(("127.0.0.1", int(sys.argv[1])), H).serve_forever()
```

Start it in the background and keep its PID so you can stop exactly that
process later:

```console
$ python3 server.py 8765 & echo $! > server.pid
```

Nothing listens on port 8766, which stands in for "the server is down".

## Reproduce: the 0-byte file

```python
# naive.py
import os, sys, urllib.request

def download(url, path):
    if os.path.exists(path):
        print("skip (already downloaded):", path)
        return
    with open(path, "wb") as f:
        with urllib.request.urlopen(url, timeout=10) as r:
            f.write(r.read())

download(sys.argv[1], sys.argv[2])
```

Everything below runs in one folder, in the order shown, with the server from
above still running. Tracebacks are shortened to their first and last lines.
One success and three kinds of failure:

```console
$ python3 naive.py http://127.0.0.1:8765/ok ok.bin; echo "exit $?"
exit 0
$ python3 naive.py http://127.0.0.1:8765/missing missing.bin; echo "exit $?"
Traceback (most recent call last):
...
urllib.error.HTTPError: HTTP Error 404: Not Found
exit 1
$ python3 naive.py http://127.0.0.1:8765/cut cut.bin; echo "exit $?"
Traceback (most recent call last):
...
http.client.IncompleteRead: IncompleteRead(1000 bytes read, 199000 more expected)
exit 1
$ python3 naive.py http://127.0.0.1:8766/ok down.bin; echo "exit $?"
Traceback (most recent call last):
...
urllib.error.URLError: <urlopen error [Errno 61] Connection refused>
exit 1
$ ls -l *.bin
-rw-r--r--  1 me  wheel       0 Oct 20 21:30 cut.bin
-rw-r--r--  1 me  wheel       0 Oct 20 21:30 down.bin
-rw-r--r--  1 me  wheel       0 Oct 20 21:30 missing.bin
-rw-r--r--  1 me  wheel  200000 Oct 20 21:30 ok.bin
```

Every failure left an empty file. The exception does propagate (`exit 1`
above), but the `with` block only closes the file; it doesn't remove it. Run
the script again:

```console
$ python3 naive.py http://127.0.0.1:8765/missing missing.bin; echo "exit $?"
skip (already downloaded): missing.bin
exit 0
```

The empty file now passes the existence check, so the script reports success.

Note the `cut` case: `r.read()` with no size raised `IncompleteRead`, so at
least the run failed loudly. The streaming version below doesn't.

## The quiet version: streaming writes a truncated file, exit 0

For big files you'd stream instead of reading everything into memory:

```python
# stream.py
import os, shutil, sys, urllib.request

def download(url, path):
    with open(path, "wb") as f:
        with urllib.request.urlopen(url, timeout=10) as r:
            shutil.copyfileobj(r, f)

download(sys.argv[1], sys.argv[2])
```

Same cut connection, saved under a new name so it doesn't hit the skip check:

```console
$ python3 stream.py http://127.0.0.1:8765/cut stream-cut.bin; echo "exit $?"
exit 0
$ ls -l stream-cut.bin
-rw-r--r--  1 me  wheel  1000 Oct 20 21:30 stream-cut.bin
```

No error, exit 0, and a 1,000-byte file where 200,000 were promised.
`copyfileobj` calls `r.read(n)` in a loop, and in my tests `read(n)` on the
cut connection returned `b''` (end of data) instead of raising, so the loop
ended normally. I got the same result with Python 3.8.2 and 3.12.13. So
`IncompleteRead` is not something to rely on: check the length yourself.

## Finding the damage you already have

Empty files are easy to find:

```console
$ find . -type f -size 0
./down.bin
./missing.bin
./cut.bin
```

`stream-cut.bin` isn't in that list: **`-size 0` only finds empty files, not
truncated ones.** For those you need to know what the size should have been.
If the server sends `Content-Length`, ask it again and compare:

```python
# verify.py — compare a downloaded file's size with the server's Content-Length
import os, sys, urllib.request

def verify(url, path):
    with urllib.request.urlopen(url, timeout=10) as r:  # headers only; body not read
        expected = r.headers.get("Content-Length")
    size = os.path.getsize(path)
    if expected is None:
        print(f"{path}: {size} bytes, server sent no Content-Length")
    elif size != int(expected):
        print(f"{path}: TRUNCATED, {size} of {expected} bytes")
    else:
        print(f"{path}: ok, {size} bytes")

verify(sys.argv[1], sys.argv[2])
```

```console
$ python3 verify.py http://127.0.0.1:8765/cut stream-cut.bin; echo "exit $?"
stream-cut.bin: TRUNCATED, 1000 of 200000 bytes
exit 0
$ python3 verify.py http://127.0.0.1:8765/ok ok.bin; echo "exit $?"
ok.bin: ok, 200000 bytes
exit 0
```

It uses GET and closes the response without reading the body, because the
test server doesn't implement HEAD. This only works if the file at that URL
hasn't changed since you downloaded it; a checksum published by the server
is better when there is one.

## The fix

Three changes:

1. **Connect before creating anything.** `urlopen` raises for a 404 or a
   refused connection before any file exists.
2. **Write to a temp file in the same folder, then `os.replace` it** to the
   real name. The real name only ever points at a complete file, so the
   "already downloaded" check stays correct.
3. **Compare the bytes written with `Content-Length`** when the server sends
   it, and delete the temp file on any failure.

```python
# fixed.py
import os, sys, tempfile, urllib.request

def download(url, path):
    if os.path.exists(path):
        print("skip (already downloaded):", path)
        return
    # Connect and check the status first: a 404 or a refused connection
    # raises here, before any file exists.
    with urllib.request.urlopen(url, timeout=10) as r:
        expected = r.headers.get("Content-Length")
        fd, tmp = tempfile.mkstemp(dir=os.path.dirname(os.path.abspath(path)),
                                   prefix=".part-")
        try:
            got = 0
            with os.fdopen(fd, "wb") as f:
                while True:
                    chunk = r.read(64 * 1024)
                    if not chunk:
                        break
                    f.write(chunk)
                    got += len(chunk)
                f.flush()
                os.fsync(f.fileno())
            # A dropped connection can end the loop early without an error.
            if expected is not None and got != int(expected):
                raise IOError(f"truncated: got {got} of {expected} bytes")
            os.chmod(tmp, 0o644)   # mkstemp creates files as 0600
            os.replace(tmp, path)  # the real name appears only when complete
        except BaseException:
            os.unlink(tmp)
            raise

download(sys.argv[1], sys.argv[2])
```

Why the `chmod`: `mkstemp` creates its file readable only by you.

```console
$ python3 -c 'import os, tempfile; fd, p = tempfile.mkstemp(dir="."); print(oct(os.stat(p).st_mode & 0o777)); os.close(fd); os.remove(p)'
0o600
```

Without it the downloads end up `0600` instead of the `-rw-r--r--` that
`open()` gave above. Pick the mode you want.

Same four cases, after clearing out the files from the naive runs:

```console
$ rm *.bin
$ python3 fixed.py http://127.0.0.1:8765/ok ok.bin; echo "exit $?"
exit 0
$ python3 fixed.py http://127.0.0.1:8765/missing missing.bin; echo "exit $?"
Traceback (most recent call last):
...
urllib.error.HTTPError: HTTP Error 404: Not Found
exit 1
$ python3 fixed.py http://127.0.0.1:8765/cut cut.bin; echo "exit $?"
Traceback (most recent call last):
...
OSError: truncated: got 1000 of 200000 bytes
exit 1
$ python3 fixed.py http://127.0.0.1:8766/ok down.bin; echo "exit $?"
Traceback (most recent call last):
...
urllib.error.URLError: <urlopen error [Errno 61] Connection refused>
exit 1
$ ls -A
fixed.py
naive.py
ok.bin
server.pid
server.py
stream.py
verify.py
```

Only the complete `ok.bin` is there (`-A` also shows dot files): no empty or partial file, and no
`.part-` file left behind. Running a failed one again tries the download
again instead of skipping it:

```console
$ python3 fixed.py http://127.0.0.1:8765/cut cut.bin; echo "exit $?"
Traceback (most recent call last):
...
OSError: truncated: got 1000 of 200000 bytes
exit 1
```

Two more details:

- **The temp file must be in the same folder** as the target. `os.replace` is
  a rename, and a rename can't cross file systems. `mkstemp(dir=...)` handles
  that.
- **No `Content-Length`** (chunked responses): the length check is skipped.
  The temp-file-and-rename part still protects you from failures that raise.

Stop the test server with the PID you saved, not by name:

```console
$ kill "$(cat server.pid)"
```

## Checklist

- An empty file with the expected name after a failed run: the file was
  opened with `"wb"` before the request succeeded.
- `exists()` skip logic + empty file = the failure is never retried. Make
  the final name appear only when the download is complete.
- Call `urlopen` first; it raises `HTTPError` (404, 500…) and `URLError`
  (`Connection refused`) before you create a file.
- Write to `tempfile.mkstemp(dir=<target folder>)`, then `os.replace`.
  Delete the temp file in `except`.
- Streaming with `shutil.copyfileobj` or `read(n)` can end on a dropped
  connection **without an error**. Count the bytes and compare with
  `Content-Length`.
- `mkstemp` files are 0600; `chmod` before the rename if others need to read
  them.
- Old damage: `find . -type f -size 0` finds the empty files only. Truncated
  ones need a size check against `Content-Length` (or a checksum).

*Tested on macOS 26.5.2 (Apple silicon) with Python 3.8.2 (`/usr/bin/python3`)
and Python 3.12.13 (`/opt/homebrew/bin/python3.12`), as reported by
`--version`, using only the standard library (`urllib.request`) against a
local `ThreadingHTTPServer` on 127.0.0.1. Outputs shown are from 3.8.2; the
naive, streaming and fixed scripts gave the same results on 3.12.13.
I did not test `requests` or other HTTP clients.*
