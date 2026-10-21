---
title: "Chrome fetch to 127.0.0.1 fails: \"Permission was denied for this request to access the loopback address space\""
description: >-
  A public web page calls a helper on 127.0.0.1 and gets "TypeError: Failed to fetch".
  The console says "Permission was denied for this request to access the `loopback`
  address space". The request never reaches your server, so no CORS header can fix it.
  The page needs the loopback-network permission. Granting local-network does not
  cover it. Reproduced with headless Chrome 155, a Python server and a small CDP script.
tags: [chrome, local-network-access, cors, localhost, javascript]
---

A page on a public site calls a helper app on your machine with
`fetch("http://127.0.0.1:8602/...")`. It used to work. Now the page gets
`TypeError: Failed to fetch`, and the console shows this:

```text
Access to fetch at 'http://127.0.0.1:8602/ping' from origin 'http://127.0.0.1:8601' has been blocked by CORS policy: Permission was denied for this request to access the `loopback` address space.
```

It says "CORS policy", but CORS headers won't fix it. Chrome's Local Network
Access check blocks the request **before it is sent**. My server logged nothing.
The page needs the user to grant it a permission. On Chrome 155, which is what I
tested, there are two separate permissions:

- `loopback-network` for `127.0.0.1` / localhost,
- `local-network` for addresses on your LAN.

Granting one does not grant the other. A page that already has `local-network`
for talking to LAN devices still can't reach `127.0.0.1`.

## Reproduce it without deploying anything

In my tests the check fired when a **public** page called a loopback or a
`local` address, and not when a loopback page called loopback (I didn't try a
`local` page calling loopback). To test this on one Mac, I used Chrome's
`--ip-address-space-overrides` switch. It makes Chrome treat
`127.0.0.1:8601` as a public site, so no real public page is needed. It
can also make `127.0.0.1:8603` count as a LAN (`local`) address.

Work in an empty folder (I used `/tmp/lna-demo`). This tiny server answers
`/ping` with CORS allowed for everyone, serves a blank page for any other path,
and logs every request it receives:

```python
# tinyhttp.py — usage: python3 -u tinyhttp.py PORT
import sys
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path.startswith("/ping"):
            body = b"pong"
            self.send_response(200)
            self.send_header("Access-Control-Allow-Origin", "*")
        else:
            body = b"<!doctype html><title>page</title><p>page</p>"
            self.send_response(200)
            self.send_header("Content-Type", "text/html")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def log_message(self, fmt, *args):
        sys.stderr.write("Origin=%s %s\n" % (self.headers.get("Origin"), fmt % args))

ThreadingHTTPServer(("127.0.0.1", int(sys.argv[1])), Handler).serve_forever()
```

The servers listen on `127.0.0.1` only. Port 8601 is the "public" page, 8602 is
the loopback helper, and 8603 plays a LAN device:

```console
$ python3 -u tinyhttp.py 8601 2>page.log & PAGE=$!
$ python3 -u tinyhttp.py 8602 2>loopback.log & LOOP=$!
$ python3 -u tinyhttp.py 8603 2>lan.log & LAN=$!
```

(`-u` matters with the system Python 3.8. Without it, stderr redirected to a file was
buffered and the logs stayed empty while I watched them.)

Start headless Chrome with a throwaway profile and the overrides. The DevTools
port is bound to `127.0.0.1`:

```console
$ CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
$ "$CHROME" --headless=new --user-data-dir=/tmp/lna-demo/profile --no-first-run \
    --remote-debugging-port=9333 \
    --ip-address-space-overrides=127.0.0.1:8601=public,127.0.0.1:8603=local \
    about:blank >chrome.log 2>&1 & CHROME_PID=$!
$ sleep 3   # give Chrome time to open the DevTools port
```

In a normal window, the user would answer a permission prompt. Headless Chrome
has no one to answer it, so this script uses the DevTools protocol instead. It
opens a new incognito-like context and grants any permissions you list
(`Browser.setPermission`). Then it loads the page, runs one expression, and
prints the console errors and the result. It uses Node's
built-in `WebSocket`; I ran it on Node 24:

```js
// cdp.mjs — usage: node cdp.mjs PAGE_URL JS_EXPRESSION [perm1,perm2]
const [, , pageURL, expr, grant] = process.argv;
const { webSocketDebuggerUrl } =
  await (await fetch("http://127.0.0.1:9333/json/version")).json();
const ws = new WebSocket(webSocketDebuggerUrl);
let nextId = 0;
const waiting = new Map();
const send = (method, params = {}, sessionId) =>
  new Promise((resolve, reject) => {
    const id = ++nextId;
    waiting.set(id, { resolve, reject });
    ws.send(JSON.stringify({ id, method, params, sessionId }));
  });
ws.onmessage = (msg) => {
  const m = JSON.parse(msg.data);
  if (m.id && waiting.has(m.id)) {
    const w = waiting.get(m.id);
    waiting.delete(m.id);
    m.error ? w.reject(new Error(m.error.message)) : w.resolve(m.result);
  } else if (m.method === "Log.entryAdded") {
    console.log("[console]", m.params.entry.level, m.params.entry.text);
  }
};
await new Promise((r) => (ws.onopen = r));

const { browserContextId } = await send("Target.createBrowserContext");
for (const name of grant ? grant.split(",") : []) {
  await send("Browser.setPermission", {
    permission: { name }, setting: "granted",
    origin: new URL(pageURL).origin, browserContextId,
  });
  console.log("granted", name);
}
const { targetId } = await send("Target.createTarget", { url: "about:blank", browserContextId });
const { sessionId } = await send("Target.attachToTarget", { targetId, flatten: true });
await send("Log.enable", {}, sessionId);
await send("Page.enable", {}, sessionId);
await send("Page.navigate", { url: pageURL }, sessionId);
await new Promise((r) => setTimeout(r, 1000));
const r = await send("Runtime.evaluate",
  { expression: expr, awaitPromise: true, returnByValue: true }, sessionId);
console.log("result:", r.result.value);
await new Promise((r) => setTimeout(r, 300));
await send("Target.disposeBrowserContext", { browserContextId });
process.exit(0);
```

### A public page calling 127.0.0.1: blocked, and the server sees nothing

```console
$ PING='fetch("http://127.0.0.1:8602/ping").then(r => r.status + " " + r.statusText, e => String(e))'
$ node cdp.mjs http://127.0.0.1:8601/ "$PING"
[console] error Access to fetch at 'http://127.0.0.1:8602/ping' from origin 'http://127.0.0.1:8601' has been blocked by CORS policy: Permission was denied for this request to access the `loopback` address space.
[console] error Failed to load resource: net::ERR_FAILED
result: TypeError: Failed to fetch
$ cat loopback.log
$
```

`loopback.log` is empty. Chrome did not send a preflight or the `GET`. Your
server can't help here: `Access-Control-Allow-Origin: *` is already on the
response, and the request never gets that far.

The page can see what is going on. Chrome 155 knows three permission names
here, and all of them are still `prompt`:

```console
$ Q='Promise.all(["local-network-access", "local-network", "loopback-network"].map(n => navigator.permissions.query({name: n}).then(s => n + "=" + s.state))).then(a => a.join(" | "))'
$ node cdp.mjs http://127.0.0.1:8601/ "$Q"
result: local-network-access=prompt | local-network=prompt | loopback-network=prompt
```

### `local-network` is not enough; `loopback-network` is

Grant each permission alone and try again:

```console
$ node cdp.mjs http://127.0.0.1:8601/ "$PING" local-network
granted local-network
[console] error Access to fetch at 'http://127.0.0.1:8602/ping' from origin 'http://127.0.0.1:8601' has been blocked by CORS policy: Permission was denied for this request to access the `loopback` address space.
[console] error Failed to load resource: net::ERR_FAILED
result: TypeError: Failed to fetch
$ node cdp.mjs http://127.0.0.1:8601/ "$PING" local-network-access
granted local-network-access
[console] error Access to fetch at 'http://127.0.0.1:8602/ping' from origin 'http://127.0.0.1:8601' has been blocked by CORS policy: Permission was denied for this request to access the `loopback` address space.
[console] error Failed to load resource: net::ERR_FAILED
result: TypeError: Failed to fetch
$ node cdp.mjs http://127.0.0.1:8601/ "$PING" loopback-network
granted loopback-network
result: 200 OK
$ cat loopback.log
Origin=http://127.0.0.1:8601 "GET /ping HTTP/1.1" 200 -
```

Chrome accepted a grant of `local-network-access`, but on 155 it did not
unblock the request.

### The other way round: a LAN address needs `local-network`

Port 8603 is overridden to `local`. That stands in for a LAN address such as a
printer or a NAS; I only tested the override, not a real LAN IP:

```console
$ LANPING='fetch("http://127.0.0.1:8603/ping").then(r => r.status + " " + r.statusText, e => String(e))'
$ node cdp.mjs http://127.0.0.1:8601/ "$LANPING" loopback-network
granted loopback-network
[console] error Access to fetch at 'http://127.0.0.1:8603/ping' from origin 'http://127.0.0.1:8601' has been blocked by CORS policy: Permission was denied for this request to access the `local` address space.
[console] error Failed to load resource: net::ERR_FAILED
result: TypeError: Failed to fetch
$ node cdp.mjs http://127.0.0.1:8601/ "$LANPING" local-network
granted local-network
result: 200 OK
```

So a page that calls both a `local` address and a loopback helper needs both
permissions. With both granted, the loopback fetch goes through:

```console
$ node cdp.mjs http://127.0.0.1:8601/ "$PING" local-network,loopback-network
granted local-network
granted loopback-network
result: 200 OK
```

### A permission that isn't granted looks like a stopped helper

Port 8609 has nothing listening. With the permission granted, the page gets the
same `TypeError` it got when the permission was not granted (state `prompt`):

```console
$ DEAD='fetch("http://127.0.0.1:8609/ping").then(r => r.status + " " + r.statusText, e => String(e))'
$ node cdp.mjs http://127.0.0.1:8601/ "$DEAD" loopback-network
granted loopback-network
[console] error Failed to load resource: net::ERR_CONNECTION_REFUSED
result: TypeError: Failed to fetch
```

The console messages are different, but the page's code can't read the console.
Inside `catch`, the two cases look the same.

### Pages that are themselves on localhost are not affected

A page served from loopback calling loopback was not blocked. No permission was
granted here:

```console
$ node cdp.mjs http://127.0.0.1:8602/ "$PING"
result: 200 OK
```

So the problem may only show up after deploy. On `localhost` during
development, everything works.

Clean up with the PIDs you saved (don't kill by name; other local servers may
match):

```console
$ kill $CHROME_PID $PAGE $LOOP $LAN
$ wait $CHROME_PID   # Chrome writes to the profile while it shuts down
$ cd /tmp && rm -rf /tmp/lna-demo
```

## What to do about it

- **On the page:** I found no way for page code to get past this, and server
  headers won't help, because the request is never sent. Before you call the helper, check
  `navigator.permissions.query({ name: "loopback-network" })`. If the fetch
  throws `TypeError`, tell the user that the site needs permission to reach apps
  on this device. Don't just show "helper not running". From the page alone, a
  permission that isn't granted and a helper that isn't running both give
  `TypeError: Failed to fetch`. `navigator.permissions.query` tells you which
  of the two it is.
- **If you're the user:** if Chrome shows a permission prompt, allow it. (I
  didn't test the prompt in a normal window, and I didn't check what the
  setting is called in Chrome's site settings.)
- **In automated browser tests:** grant `loopback-network` (and `local-network`
  if you call LAN addresses) for the page's origin. `Browser.setPermission`
  above does that over CDP.
- **Don't confuse the two:** an error that says `` `loopback` address space ``
  needs `loopback-network`. One that says `` `local` address space `` needs
  `local-network`.

## Checklist

- Console says `Permission was denied for this request to access the
  `loopback` address space` → Chrome blocked it before sending. Your server
  logs show nothing.
- CORS headers on the helper don't change this.
- `127.0.0.1` → `loopback-network`. An address treated as `local` (here via
  the override) → `local-network`.
  Each is granted separately. Granting `local-network-access` did not help on
  Chrome 155.
- A loopback page calling loopback wasn't blocked, so test from a real public page
  or with `--ip-address-space-overrides=HOST:PORT=public`.
- A permission that isn't granted and a stopped helper both throw `TypeError: Failed to fetch`.
  Use `navigator.permissions.query` to tell them apart.

Not tested: what the permission prompt looks like in a normal window, an `https`
public page calling an `http` loopback URL (mixed content), real LAN IPs rather
than the override, Chrome versions other than 155, other browsers, and
Playwright's `grantPermissions`.

*Tested on macOS 26.5.2 (Apple silicon) with Google Chrome 155.0.8059.85
(`--headless=new`, throwaway profile in `/tmp`), Node v24.14.1 and the system
Python 3.8.2, in zsh 5.9. All three test servers (ports 8601, 8602, 8603) listened on 127.0.0.1 only.*
