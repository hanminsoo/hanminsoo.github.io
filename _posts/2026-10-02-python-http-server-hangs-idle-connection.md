---
title: "Python HTTPServer hangs: one idle connection blocks every request (use ThreadingHTTPServer)"
description: >-
  A small Python server built on http.server.HTTPServer stopped answering — curl
  timed out with exit code 28 — while one client held a TCP connection open without
  sending anything. Reproduced in 20 lines, then fixed with ThreadingHTTPServer
  or a per-connection timeout.
tags: [python, http, macos]
---

A Python server built on `http.server.HTTPServer` can **stop answering** without
a traceback or a log line: requests hang until the client gives up
(`curl: (28) Operation timed out`), and then everything works again.

I reproduced this on my Mac. `HTTPServer` handles **one connection at a time**. If a
client opens a TCP connection and doesn't send a request on it — for example a
browser's speculative connection (preconnect) — the server waits on that connection,
and every other request queues behind it. (I simulated the idle client with a Python
socket below; I did not reproduce it with a browser.)

## Reproduce it in two files

A minimal server. With `--threading` it uses `ThreadingHTTPServer`, otherwise the
plain `HTTPServer`:

```python
# server.py
import sys
from http.server import HTTPServer, ThreadingHTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        body = b"ok\n"
        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

server_class = ThreadingHTTPServer if "--threading" in sys.argv else HTTPServer
server_class(("127.0.0.1", 8765), Handler).serve_forever()
```

And a client that simulates a preconnect: it connects and then says nothing.

```python
# preconnect.py
# Open a TCP connection and send nothing (simulates an idle preconnect).
import socket, time
s = socket.create_connection(("127.0.0.1", 8765))
time.sleep(30)
```

Run the plain server, make one normal request, then open the idle connection and try
again:

```console
$ python3 server.py &
$ curl -s -m 5 -w '%{http_code} %{time_total}s\n' http://127.0.0.1:8765/
ok
200 0.002656s
$ python3 preconnect.py &
$ sleep 0.5
$ curl -s -m 5 -w '%{http_code} %{time_total}s\n' http://127.0.0.1:8765/; echo "curl exit $?"
000 5.112399s
curl exit 28
$ kill %2      # close the idle connection
$ curl -s -m 5 -w '%{http_code} %{time_total}s\n' http://127.0.0.1:8765/
ok
200 0.001716s
```

The second request never got an answer. As soon as the idle connection went away,
the server was healthy again — so the hang can look intermittent from the outside.

## Where the server is stuck

While the request hangs, `sample` shows the server's main thread blocked in a socket
`recv` (the full output is long; these are the lines around `recv`):

```console
$ sample <server-pid> 1
...
    776 sock_recv_into  (in _socket.cpython-38-darwin.so) + 268
    776 sock_call_ex  (in _socket.cpython-38-darwin.so) + 368
    776 sock_recv_impl  (in _socket.cpython-38-darwin.so) + 36
    776 __recvfrom  (in libsystem_kernel.dylib) + 8
```

The stack doesn't say *which* connection that `recv` is on. My reading: `HTTPServer`
accepts a connection and handles it to completion before calling `accept()` again,
and `BaseHTTPRequestHandler.timeout` is `None` by default, so "to completion" means
"until the client sends something or hangs up". The idle connection was accepted
first, so the thread is most likely waiting for its request line, while `curl` waits
in the listen backlog. That fits the fact that closing the idle connection
immediately unblocked the server.

## Fix 1: use ThreadingHTTPServer

`http.server.ThreadingHTTPServer` (Python 3.7+) is `HTTPServer` with
`socketserver.ThreadingMixIn`: one thread per connection, so an idle connection only
blocks its own thread. It is a one-word change:

Stop the previous server and the idle client first (Ctrl-C them, or `kill` the PIDs
the shell printed when you started them).

```console
$ python3 server.py --threading &
$ python3 preconnect.py &
$ sleep 0.5
$ curl -s -m 5 -w '%{http_code} %{time_total}s\n' http://127.0.0.1:8765/; echo "curl exit $?"
ok
200 0.001708s
curl exit 0
```

Per the Python docs its `daemon_threads` is `True`, so threads stuck on idle connections
shouldn't keep the process alive when you stop the server (I didn't test that part).

`python3 -m http.server` already uses `ThreadingHTTPServer` (since 3.7), so the
command-line file server doesn't show this problem — the same test returned `200`
immediately. It bites when you write `HTTPServer(...)` in your own code, which is
what many examples and older snippets do.

## Fix 2 (if you must stay single-threaded): a per-connection timeout

If the handler isn't safe to run in parallel, at least stop one connection from
holding the server forever. Setting `timeout` on the handler applies a socket timeout
to each connection; an idle one is dropped after that many seconds:

```python
# server_timeout.py
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    timeout = 2   # seconds to wait for a request line before dropping the connection

    def do_GET(self):
        body = b"ok\n"
        self.send_response(200)
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

HTTPServer(("127.0.0.1", 8765), Handler).serve_forever()
```

Stop the previous server and the idle client first (Ctrl-C them, or `kill` the PIDs
the shell printed when you started them).

```console
$ python3 server_timeout.py &
$ python3 preconnect.py &
$ sleep 0.5
$ curl -s -m 5 -w '%{http_code} %{time_total}s\n' http://127.0.0.1:8765/; echo "curl exit $?"
ok
200 1.448683s
curl exit 0
```

The request now succeeds, but only after the idle connection's timeout runs out
(the idle connection was opened about half a second before `curl` — the `sleep 0.5`
above — so the wait was ~1.5 s rather than the full 2 s).

With Python 3.12 the server's stderr log shows the order of events — the idle
connection is dropped first, then `curl`'s request is served:

```console
127.0.0.1 - - [02/Oct/2026 10:04:48] Request timed out: TimeoutError('timed out')
127.0.0.1 - - [02/Oct/2026 10:04:48] "GET / HTTP/1.1" 200 -
```

Every idle connection costs every other client up to `timeout` seconds — fine
for a script, not for a page that loads several resources. Prefer Fix 1; you can
combine both.

## Checklist

- A Python HTTP server that intermittently stops answering, with no error in its log:
  check whether it's `HTTPServer` (single connection at a time).
- Reproduce by opening a TCP connection that sends nothing, then making a request.
- Switch to `ThreadingHTTPServer` — same constructor, same handler.
- If you need serial handling, set `timeout` on the handler so an idle connection
  can't hold the server forever.
- `sample <pid>` showing the server's only thread blocked in `recv` is a strong hint.

*Tested on macOS 26.5.2 (Apple silicon) with Python 3.8.2 (`/usr/bin/python3`,
Command Line Tools) and Python 3.12.13 (Homebrew), curl 8.7.1. Outputs above are
from 3.8.2 unless noted; the server log lines were checked on 3.12. Server bound to `127.0.0.1`
only. The idle client was a Python socket, not a browser.*
