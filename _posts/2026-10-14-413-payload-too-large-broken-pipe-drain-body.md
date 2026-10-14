---
title: "Server returns 413 but the client gets BrokenPipeError / ECONNRESET: drain the body first"
description: >-
  A Python server that rejects oversized uploads with 413 looked fine in curl and
  Chrome, but Python clients failed with "BrokenPipeError: [Errno 32] Broken pipe"
  or "ConnectionResetError: [Errno 54]", and Node's fetch with EPIPE/ECONNRESET.
  The server replied and closed before reading the body. Reproduced with a small
  http.server script, then fixed by draining (up to a cap) before sending 413.
tags: [python, http, node, macos]
---

Your server checks `Content-Length`, sees the upload is too big, and answers
`413 Payload Too Large`. In `curl` you see the 413. But a Python client gets
`BrokenPipeError: [Errno 32] Broken pipe` or `ConnectionResetError: [Errno 54]
Connection reset by peer`, and Node's `fetch` throws `TypeError: fetch failed`
with `EPIPE` or `ECONNRESET` — the client never sees the 413 you sent.

I reproduced this on my Mac. The server sent the 413 and **closed the connection
without reading the request body**. The client was still writing the body. When a socket is
closed with unread data in its receive buffer, TCP answers with a reset (RST) rather
than a normal close, and the client's next `send()` or `recv()` fails. (I didn't
capture packets; the client-side errors below fit that.) Clients that read the response *while* uploading (curl,
Chrome) pick up the 413 in time; clients that finish writing before reading
(Python's `http.client`/`urllib`, and Node's `fetch` in some runs) don't.

## Reproduce it

A server that accepts bodies up to 1 MB. Without `--drain` it answers 413 straight
away; with `--drain` it reads and discards the body first (up to a cap — more on
that below):

```python
# server.py
import sys
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler

LIMIT = 1_000_000          # largest body we accept
MAX_DRAIN = 50_000_000     # largest body we'll read just to throw it away
DRAIN = "--drain" in sys.argv

class Handler(BaseHTTPRequestHandler):
    timeout = 10           # a client that stops sending can't hold a thread forever

    def do_POST(self):
        length = int(self.headers.get("Content-Length", 0))
        if length > LIMIT:
            if DRAIN and length <= MAX_DRAIN:
                self.drain(length)
            self.reply(413, b"too large\n")
            return
        self.rfile.read(length)
        self.reply(200, b"ok\n")

    def drain(self, length):
        # read and discard the body so the client can finish sending
        while length > 0:
            chunk = self.rfile.read(min(length, 65536))
            if not chunk:
                break
            length -= len(chunk)

    def reply(self, code, body):
        self.send_response(code)
        self.send_header("Content-Type", "text/plain")
        self.send_header("Content-Length", str(len(body)))
        self.send_header("Connection", "close")
        self.end_headers()
        self.wfile.write(body)
        self.close_connection = True

ThreadingHTTPServer(("127.0.0.1", 8766), Handler).serve_forever()
```

A client using the standard library, posting N bytes:

```python
# client.py
import http.client, sys
body = b"x" * int(sys.argv[1])
conn = http.client.HTTPConnection("127.0.0.1", 8766)
try:
    conn.request("POST", "/", body=body, headers={"Content-Type": "application/octet-stream"})
    r = conn.getresponse()
    print(r.status, r.read())
except Exception as e:
    print(type(e).__name__, e)
```

Start the server without `--drain`, send a small body, then a 20 MB one five times:

```console
$ python3 server.py &
$ python3 client.py 100
200 b'ok\n'
$ for i in 1 2 3 4 5; do python3 client.py 20000000; done | sort | uniq -c
   4 BrokenPipeError [Errno 32] Broken pipe
   1 ConnectionResetError [Errno 54] Connection reset by peer
```

Not one 413. Which error you get depends on timing — whether the reset arrives while
the client is still writing (`EPIPE`) or when it reads (`ECONNRESET`). Across several
runs of five I saw both, in varying proportions (one run was five `BrokenPipeError`s),
but never a 413.

`urllib.request` is built on `http.client` and fails the same way — you get a
`URLError`, not the `HTTPError` with `.code == 413` your code probably handles:

```console
$ python3 -c '
import urllib.request, urllib.error
try:
    urllib.request.urlopen(urllib.request.Request("http://127.0.0.1:8766/", data=b"x" * 20_000_000))
except urllib.error.HTTPError as e:
    print("HTTPError", e.code)
except Exception as e:
    print(type(e).__name__, e)'
URLError <urlopen error [Errno 32] Broken pipe>
```

Node's built-in `fetch` failed in two to four of every five tries (one run shown):

```js
// upload.mjs
try {
  const r = await fetch("http://127.0.0.1:8766/", { method: "POST", body: new Uint8Array(20_000_000) });
  console.log(r.status, (await r.text()).trim());
} catch (e) { console.log(String(e), e.cause?.code); }
```

```console
$ for i in 1 2 3 4 5; do node upload.mjs; done | sort | uniq -c
   2 413 too large
   1 TypeError: fetch failed ECONNRESET
   1 TypeError: fetch failed EPIPE
   1 TypeError: terminated EPIPE
```

## Why curl says everything is fine

The same server, the same 20 MB, with curl:

```console
$ head -c 20000000 /dev/zero > big.bin
$ curl -s -o /dev/null -w '%{http_code}\n' --data-binary @big.bin http://127.0.0.1:8766/
413
```

`curl -v` shows why (excerpt): for a large body curl sends `Expect: 100-continue`
and waits before uploading, so the 413 arrives before any body is sent:

```console
> Content-Length: 20000000
> Content-Type: application/x-www-form-urlencoded
> Expect: 100-continue
>
* HTTP 1.0, assume close after body
< HTTP/1.0 413 Request Entity Too Large
```

Even with `-H 'Expect:'` (upload immediately) curl got the 413 in five of five
tries — it watches for a response while sending. Headless Chrome (`fetch` from a page
on the same server) also got `413` five times out of five. So if you only test with
curl or a browser, the bug is invisible; it shows up when a script or another
service is the client.

## Fix: read (drain) the body, then send 413

Stop the previous server first (Ctrl-C, or `kill` the PID the shell printed when you
started it), then start it with `--drain`:

```console
$ python3 server.py --drain &
$ for i in 1 2 3 4 5; do python3 client.py 20000000; done | sort | uniq -c
   5 413 b'too large\n'
$ for i in 1 2 3 4 5; do node upload.mjs; done | sort | uniq -c
   5 413 too large
```

and the `urllib` snippet above now prints `HTTPError 413`. Draining 20 MB over
loopback took about 50 ms for the whole client run here.

Once the server has read the whole body there is no unread data when it closes, so
the close should be a normal one instead of a reset, and the client reads the 413
after it finishes writing — which is what all four clients did.

### Don't drain without limits

Draining means you still receive the bytes you wanted to refuse. Two guards in
`server.py` above:

- **`MAX_DRAIN`**: above it, don't drain — answer and close, and accept that this
  client will see a reset. A client claiming a 10 GB body shouldn't get to make you
  read 10 GB. With `--drain`, a 60 MB body (over the 50 MB cap) gave the same
  `BrokenPipeError` / `ConnectionResetError` as before.
- **`timeout`** on the handler: a client that sends the headers and then trickles
  (or stops) can't hold a thread forever. With a client that sent the headers for a
  2 MB body plus 100 bytes and then went quiet, the server gave up and closed the
  connection after 10 s (without a response — the client just sees the close).

Pick `MAX_DRAIN` a few times above your real limit — big enough that honest clients
who are slightly over get a readable 413.

## Checklist

- A client reports `Broken pipe` / `EPIPE` / `ECONNRESET` on upload, but the server
  log says it sent `413`: the server probably closed without reading the body (a proxy can cause the same symptom).
- Test with a client that writes the whole body before reading — Python
  `http.client`/`urllib`, Node `fetch`. curl and browsers hide the problem.
- Before answering 413 (and likely any early error on a request with a body; I only tested 413), read and
  discard the remaining `Content-Length` bytes.
- Cap the drain and set a socket timeout; past the cap, closing (and the reset) is
  the trade-off I chose.
- This server only handles `Content-Length` bodies; a chunked upload needs the same
  idea applied to the chunked stream.

*Tested on macOS 26.5.2 (Apple silicon) with Python 3.8.2 (`/usr/bin/python3`,
Command Line Tools) and Python 3.12.13 (Homebrew; same results with `client.py`),
Node 24.14.1, curl 8.7.1, Google Chrome 154.0.8037.99 (headless, fresh profile).
Server bound to `127.0.0.1` only, so all traffic was loopback; I didn't test over a
real network, where timing — and which error you see — may differ.*
