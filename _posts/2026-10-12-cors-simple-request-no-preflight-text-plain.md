---
title: "CORS blocked the response but the POST still ran: text/plain skips the preflight (require application/json)"
description: >-
  A cross-origin fetch failed with "TypeError: Failed to fetch", yet the local API
  had already written the data. A POST with Content-Type text/plain is a CORS
  "simple request": the browser sends it without an OPTIONS preflight and only
  blocks reading the response. Reproduced with headless Chrome and a 35-line Python
  server, then fixed by rejecting writes that aren't application/json (415).
tags: [cors, http, python, chrome, security]
---

A page on another origin calls your local API, the browser console says
`TypeError: Failed to fetch` (CORS error) — and the write **happened anyway**. Your
server logged the `POST`, the data is in the store, and only the *response* was
hidden from the page.

I reproduced this on my Mac with headless Chrome. The cause: a `POST` whose
`Content-Type` is `text/plain` (or `application/x-www-form-urlencoded` or
`multipart/form-data`) is a CORS **simple request**. The browser sends it straight
to the server **without an `OPTIONS` preflight**, and CORS is only checked
afterwards, when the page tries to read the response. Your `OPTIONS` handler —
however strict — never gets a say.

## Reproduce it

Work in an empty folder (I used `/tmp/cors-demo`). A small JSON API on port 8766 that is meant to accept writes only from
`http://127.0.0.1:8765`. With `--strict` it adds the fix shown later; ignore that
for now.

```python
# api.py — a local JSON API on 127.0.0.1:8766 that only allows one origin
import sys
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler

ALLOWED_ORIGIN = "http://127.0.0.1:8765"
STRICT = "--strict" in sys.argv

class Handler(BaseHTTPRequestHandler):
    def do_OPTIONS(self):
        # CORS preflight: only the allowed origin gets the green light
        self.send_response(204)
        if self.headers.get("Origin") == ALLOWED_ORIGIN:
            self.send_header("Access-Control-Allow-Origin", ALLOWED_ORIGIN)
            self.send_header("Access-Control-Allow-Methods", "POST")
            self.send_header("Access-Control-Allow-Headers", "Content-Type")
        self.end_headers()

    def do_POST(self):
        length = int(self.headers.get("Content-Length", 0))
        body = self.rfile.read(length)
        ctype = self.headers.get("Content-Type", "")
        if STRICT and ctype.split(";")[0].strip().lower() != "application/json":
            self.send_response(415)
            self.send_header("Content-Length", "0")
            self.end_headers()
            return
        print(f"WRITE from Origin={self.headers.get('Origin')} "
              f"Content-Type={ctype!r} body={body!r}", flush=True)
        self.send_response(200)
        if self.headers.get("Origin") == ALLOWED_ORIGIN:
            self.send_header("Access-Control-Allow-Origin", ALLOWED_ORIGIN)
        self.send_header("Content-Length", "0")
        self.end_headers()

ThreadingHTTPServer(("127.0.0.1", 8766), Handler).serve_forever()
```

The `WRITE` line stands in for "the data was saved".

A page served from a **different** origin (port 8767) that sends the same JSON
body twice — once labelled `application/json`, once labelled `text/plain`. Save it
as `evil/index.html` (`mkdir evil` first):

```html
<!-- evil/index.html — served from a different origin (port 8767) -->
<script>
const url = "http://127.0.0.1:8766/notes";
const log = (m) => document.body.insertAdjacentHTML("beforeend", `<p>${m}</p>`);

// 1. JSON content type: not a "simple" request, so the browser preflights first
fetch(url, { method: "POST",
             headers: { "Content-Type": "application/json" },
             body: '{"title":"from json"}' })
  .then(r => log("json: " + r.status), e => log("json: " + e));

// 2. text/plain: a "simple" request, sent straight away with no preflight
fetch(url, { method: "POST",
             headers: { "Content-Type": "text/plain" },
             body: '{"title":"from text/plain"}' })
  .then(r => log("text: " + r.status), e => log("text: " + e));
</script>
```

Start the API and serve the page, keeping each PID so you can stop exactly these
processes later. `python3 -u` keeps the server's log unbuffered so you see it
immediately:

```console
$ python3 -u api.py > api.log 2>&1 & API=$!
$ python3 -m http.server 8767 --bind 127.0.0.1 --directory evil > /dev/null 2>&1 & EVIL=$!
```

Then load the page in headless Chrome with a throwaway profile in `/tmp` (opening
the URL in normal Chrome and watching the console works too). On my machine
headless Chrome sometimes printed the DOM and then didn't exit, so I run it in the
background, give it five seconds, and stop it by its PID:

```console
$ CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
$ "$CHROME" --headless --user-data-dir=/tmp/cors-demo-profile --no-first-run \
    --virtual-time-budget=3000 --dump-dom http://127.0.0.1:8767/ > dom.html 2>/dev/null & TAB=$!
$ sleep 5; kill $TAB 2>/dev/null; grep -o '<body>.*</body>' dom.html
<body><p>text: TypeError: Failed to fetch</p><p>json: TypeError: Failed to fetch</p></body>
```

(The two `<p>` lines can come out in either order.)

From the page's point of view, both requests failed the same way. The server log
tells a different story:

```console
$ cat api.log
127.0.0.1 - - [12/Oct/2026 10:10:12] "OPTIONS /notes HTTP/1.1" 204 -
WRITE from Origin=http://127.0.0.1:8767 Content-Type='text/plain' body=b'{"title":"from text/plain"}'
127.0.0.1 - - [12/Oct/2026 10:10:12] "POST /notes HTTP/1.1" 200 -
```

- The `application/json` request was preflighted. The `OPTIONS` answer had no
  `Access-Control-Allow-Origin` for port 8767, so Chrome **never sent the `POST`**.
- The `text/plain` request had **no preflight**. The `POST` reached the handler and
  the write ran. Chrome then saw no `Access-Control-Allow-Origin` on the `200` and
  hid the response from the page. In this test, that was all CORS did for the
  simple request: it hid the answer, it didn't stop the write.

The body was JSON both times. The server parsed whatever arrived; only the label
differed, and the label decides whether there's a preflight.

Stop both servers before the next step (port 8766 has to be free for the fixed
server):

```console
$ kill $API $EVIL
```

## Fix: refuse writes that aren't `application/json`

`application/json` is not one of the three content types a simple request may
carry, so a browser has to preflight it — and the preflight is where your origin
check actually stops a request. Make that the only way in: reject any write whose
`Content-Type` isn't `application/json`, **before** processing the request. (`api.py`
reads the body before the check, but on a `415` it doesn't use or store it.)
That is the `STRICT` branch in `api.py`:

```python
if STRICT and ctype.split(";")[0].strip().lower() != "application/json":
    self.send_response(415)
```

It compares only the media type, so `application/json; charset=utf-8` still
passes. Same test with `--strict`. This time I also serve a copy of the same page
from the allowed origin (port 8765), to check the legitimate client still works:

```console
$ mkdir good && cp evil/index.html good/
$ python3 -u api.py --strict > api.log 2>&1 & API=$!
$ python3 -m http.server 8767 --bind 127.0.0.1 --directory evil > /dev/null 2>&1 & EVIL=$!
$ python3 -m http.server 8765 --bind 127.0.0.1 --directory good > /dev/null 2>&1 & GOOD=$!
```

Load the page from the other origin (8767), then from the allowed origin (8765):

```console
$ rm -rf /tmp/cors-demo-profile
$ "$CHROME" --headless --user-data-dir=/tmp/cors-demo-profile --no-first-run \
    --virtual-time-budget=3000 --dump-dom http://127.0.0.1:8767/ > dom.html 2>/dev/null & TAB=$!
$ sleep 5; kill $TAB 2>/dev/null; grep -o '<body>.*</body>' dom.html
<body><p>json: TypeError: Failed to fetch</p><p>text: TypeError: Failed to fetch</p></body>
$ rm -rf /tmp/cors-demo-profile
$ "$CHROME" --headless --user-data-dir=/tmp/cors-demo-profile --no-first-run \
    --virtual-time-budget=3000 --dump-dom http://127.0.0.1:8765/ > dom.html 2>/dev/null & TAB=$!
$ sleep 5; kill $TAB 2>/dev/null; grep -o '<body>.*</body>' dom.html
<body><p>text: TypeError: Failed to fetch</p><p>json: 200</p></body>
```

```console
$ cat api.log
127.0.0.1 - - [12/Oct/2026 10:10:20] "OPTIONS /notes HTTP/1.1" 204 -
127.0.0.1 - - [12/Oct/2026 10:10:20] "POST /notes HTTP/1.1" 415 -
127.0.0.1 - - [12/Oct/2026 10:10:25] "POST /notes HTTP/1.1" 415 -
127.0.0.1 - - [12/Oct/2026 10:10:25] "OPTIONS /notes HTTP/1.1" 204 -
WRITE from Origin=http://127.0.0.1:8765 Content-Type='application/json' body=b'{"title":"from json"}'
127.0.0.1 - - [12/Oct/2026 10:10:25] "POST /notes HTTP/1.1" 200 -
```

The `text/plain` writes now get `415` and no `WRITE` line, from either origin. The
only write that ran is the preflighted JSON one from the allowed origin.

For comparison, when I loaded the 8765 page against the server *without*
`--strict`, its `text/plain` request also wrote (and got `200`) — so if your own front end sends `text/plain`, for example
to avoid a preflight round trip, switch it to `application/json` before turning
the check on.

Checking the header directly with curl (curl isn't a browser and doesn't do CORS;
this only shows the server's answer):

```console
$ curl -s -o /dev/null -w '%{http_code}\n' -X POST -H 'Content-Type: text/plain' -d '{}' http://127.0.0.1:8766/notes
415
$ curl -s -o /dev/null -w '%{http_code}\n' -X POST -H 'Content-Type: application/json; charset=utf-8' -d '{}' http://127.0.0.1:8766/notes
200
```

Stop the three servers by the PIDs you saved, and delete the throwaway profile:

```console
$ kill $API $EVIL $GOOD
$ rm -rf /tmp/cors-demo-profile
```

## Checklist

- "CORS error" in the browser does **not** mean the request didn't reach the
  server. For simple requests it only means the page can't read the response.
- Simple request = `GET`/`HEAD`/`POST` with only basic headers and a
  `Content-Type` of `text/plain`, `application/x-www-form-urlencoded` or
  `multipart/form-data`. These are sent without a preflight.
- A strict `OPTIONS` handler protects nothing against those. Check the server log
  for a `POST` with no `OPTIONS` before it.
- On write endpoints, require `Content-Type: application/json` (compare the media
  type, ignore parameters) and answer `415` otherwise, before processing the request.
- Make sure your own client sends `application/json`.
- Keep checking `Origin` in the preflight; the content-type rule only makes sure the
  preflight happens.

*Tested on macOS 26.5.2 (Apple silicon) with Google Chrome 154.0.8037.99 (headless,
fresh profile), Python 3.8.2 (`/usr/bin/python3`, Command Line Tools), curl 8.7.1.
All servers bound to `127.0.0.1`; both origins were local ports on the same machine.
I did not test a public website calling localhost, or other browsers.*
