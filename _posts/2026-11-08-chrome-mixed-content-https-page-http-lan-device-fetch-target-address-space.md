---
title: "Chrome \"Mixed Content: ... requested an insecure resource\" from an https page to a LAN device: fetch with targetAddressSpace: \"local\""
description: >-
  An https page calls a device on your network over plain http, like a printer or a NAS, and gets
  "TypeError: Failed to fetch" with "Mixed Content: ... This request has been blocked".
  The local-network permission doesn't help on its own. On Chrome 156 the request went through
  when it was sent with fetch(url, { targetAddressSpace: "local" }) and the permission was granted.
  In these tests on Chrome 156, http://127.0.0.1 and localhost were not blocked as mixed content. Reproduced with headless Chrome,
  a Python server and a small CDP script.
tags: [chrome, local-network-access, mixed-content, fetch, javascript]
---

Your site is served over `https`. It talks to a device on the user's network
that only speaks plain `http`, such as a printer, a NAS or a home-made
controller, at an address like `http://printer.test:8603`. The page gets
`TypeError: Failed to fetch`, and the console says:

```text
Mixed Content: The page at 'https://example.com/' was loaded over HTTPS, but requested an insecure resource 'http://printer.test:8603/ping'. This request has been blocked; the content must be served over HTTPS.
```

Granting the site the `local-network` permission doesn't change this. In my
tests on Chrome 156 the request only went through when **both** of these were true:

1. the page sent it with `fetch(url, { targetAddressSpace: "local" })`, and
2. the `local-network` permission was granted.

A helper on `http://127.0.0.1` or `http://localhost` is different. Chrome did
not block it as mixed content. It only needed the `loopback-network` permission
([earlier post](/2026/10/chrome-fetch-127-0-0-1-permission-denied-loopback-address-space/)).

## Reproduce it on one Mac

I didn't use a real LAN device. Two Chrome switches fake one on `127.0.0.1`:

- `--host-resolver-rules="MAP printer.test 127.0.0.1"` gives it a name that
  is not `localhost` or an IP literal. Chrome treats `localhost` and
  `127.0.0.1` as trustworthy and won't block them as mixed content, so the
  name is what makes it count as a normal insecure URL.
- `--ip-address-space-overrides=127.0.0.1:8603=local` makes Chrome treat
  port 8603 as a LAN (`local`) address instead of loopback.

The page is the real `https://example.com/`, so it is a genuinely public https
origin. Only the fetches go to this Mac.

Work in an empty folder (I used `/tmp/lna-mixed`). This server answers `/ping`
with CORS allowed for everyone and logs every request it receives:

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

Port 8602 is a loopback helper and 8603 plays the LAN device. Both listen on
`127.0.0.1` only:

Save `tinyhttp.py` (above) and `cdp.mjs` (below) in `/tmp/lna-mixed`, and
start the servers and Chrome from inside that folder:

```console
$ mkdir -p /tmp/lna-mixed && cd /tmp/lna-mixed
$ python3 -u tinyhttp.py 8602 2>loopback.log & LOOP=$!
$ python3 -u tinyhttp.py 8603 2>lan.log & LAN=$!
$ CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
$ "$CHROME" --headless=new --user-data-dir=/tmp/lna-mixed/profile --no-first-run \
    --remote-debugging-port=9333 \
    --host-resolver-rules="MAP printer.test 127.0.0.1" \
    --ip-address-space-overrides=127.0.0.1:8603=local \
    about:blank >chrome.log 2>&1 & CHROME_PID=$!
$ sleep 3   # give Chrome time to open the DevTools port
```

Headless Chrome can't show a permission prompt, so this script grants
permissions over the DevTools protocol (`Browser.setPermission`) in a fresh
browser context. Then it opens the page, runs one expression, and prints the
console messages and the result. It uses Node's built-in `WebSocket` (Node 24):

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

### A plain fetch to the LAN device: mixed content, with or without the permission

```console
$ LAN_PING='fetch("http://printer.test:8603/ping").then(r => r.status + " " + r.statusText, e => String(e))'
$ node cdp.mjs https://example.com/ "$LAN_PING"
[console] error Mixed Content: The page at 'https://example.com/' was loaded over HTTPS, but requested an insecure resource 'http://printer.test:8603/ping'. This request has been blocked; the content must be served over HTTPS.
result: TypeError: Failed to fetch
$ node cdp.mjs https://example.com/ "$LAN_PING" local-network
granted local-network
[console] error Mixed Content: The page at 'https://example.com/' was loaded over HTTPS, but requested an insecure resource 'http://printer.test:8603/ping'. This request has been blocked; the content must be served over HTTPS.
result: TypeError: Failed to fetch
$ cat lan.log
$
```

The server saw nothing. Going by the messages, the mixed-content block comes
first: with the permission granted or not, the only error is the mixed-content
one, and there is no message about the address space or the permission.

### The same fetch with `targetAddressSpace: "local"`

```console
$ LAN_TAS='fetch("http://printer.test:8603/ping", {targetAddressSpace: "local"}).then(r => r.status + " " + r.statusText, e => String(e))'
$ node cdp.mjs https://example.com/ "$LAN_TAS"
[console] warning Mixed Content: The page at 'https://example.com/' was loaded over HTTPS, but requested an insecure resource 'http://printer.test:8603/ping'. This content should also be served over HTTPS.
[console] error Access to fetch at 'http://printer.test:8603/ping' from origin 'https://example.com' has been blocked by CORS policy: Permission was denied for this request to access the `local` address space.
[console] error Failed to load resource: net::ERR_FAILED
result: TypeError: Failed to fetch
$ node cdp.mjs https://example.com/ "$LAN_TAS" local-network
granted local-network
[console] warning Mixed Content: The page at 'https://example.com/' was loaded over HTTPS, but requested an insecure resource 'http://printer.test:8603/ping'. This content should also be served over HTTPS.
result: 200 OK
$ cat lan.log
Origin=https://example.com "GET /ping HTTP/1.1" 200 -
```

With the option, the mixed-content message went from an `error` ("has been
blocked") to a `warning` ("should also be served"). After that, the request hit
the normal Local Network Access check. Without the permission it failed with
`` `local` address space ``, and once the permission was granted it went through.
The warning still shows up even when the request works.

### The option has to name the right address space

`printer.test:8602` resolves to loopback, because there's no override for that
port. If you tell Chrome it is `local`, Chrome refuses, even with both
permissions granted. `"loopback"` works:

```console
$ WRONG='fetch("http://printer.test:8602/ping", {targetAddressSpace: "local"}).then(r => r.status + " " + r.statusText, e => String(e))'
$ node cdp.mjs https://example.com/ "$WRONG" local-network,loopback-network
granted local-network
granted loopback-network
[console] warning Mixed Content: The page at 'https://example.com/' was loaded over HTTPS, but requested an insecure resource 'http://printer.test:8602/ping'. This content should also be served over HTTPS.
[console] error Access to fetch at 'http://printer.test:8602/ping' from origin 'https://example.com' has been blocked by CORS policy: Request had a target IP address space of `local` yet the resource is in address space `loopback`.
[console] error Failed to load resource: net::ERR_FAILED
result: TypeError: Failed to fetch
$ RIGHT='fetch("http://printer.test:8602/ping", {targetAddressSpace: "loopback"}).then(r => r.status + " " + r.statusText, e => String(e))'
$ node cdp.mjs https://example.com/ "$RIGHT" loopback-network
granted loopback-network
[console] warning Mixed Content: The page at 'https://example.com/' was loaded over HTTPS, but requested an insecure resource 'http://printer.test:8602/ping'. This content should also be served over HTTPS.
result: 200 OK
```

Going by the error message, Chrome compares the address space it actually
connected to with the one you named. So the option is not a general way to
switch off mixed content for any http URL. Use the value that matches where the
device really is.

`"public"` was not rejected as a bad value. The fetch just behaved as if there
was no option, and was blocked as mixed content:

```console
$ PUB='fetch("http://printer.test:8603/ping", {targetAddressSpace: "public"}).then(r => r.status + " " + r.statusText, e => String(e))'
$ node cdp.mjs https://example.com/ "$PUB" local-network
granted local-network
[console] error Mixed Content: The page at 'https://example.com/' was loaded over HTTPS, but requested an insecure resource 'http://printer.test:8603/ping'. This request has been blocked; the content must be served over HTTPS.
result: TypeError: Failed to fetch
```

### 127.0.0.1 and localhost: no mixed content, only the permission

```console
$ LOOP_PING='fetch("http://127.0.0.1:8602/ping").then(r => r.status + " " + r.statusText, e => String(e))'
$ node cdp.mjs https://example.com/ "$LOOP_PING"
[console] error Access to fetch at 'http://127.0.0.1:8602/ping' from origin 'https://example.com' has been blocked by CORS policy: Permission was denied for this request to access the `loopback` address space.
[console] error Failed to load resource: net::ERR_FAILED
result: TypeError: Failed to fetch
$ node cdp.mjs https://example.com/ "$LOOP_PING" loopback-network
granted loopback-network
result: 200 OK
$ node cdp.mjs https://example.com/ 'fetch("http://localhost:8602/ping").then(r => r.status + " " + r.statusText, e => String(e))' loopback-network
granted loopback-network
result: 200 OK
```

There was no mixed-content message at all, not even a warning. So an https
page talking to `http://127.0.0.1` needs no `targetAddressSpace`. Only a name
or address that isn't `localhost`/`127.0.0.1` runs into mixed content.

### Feature detection

Chrome 156 exposes the option on `Request`:

```console
$ node cdp.mjs https://example.com/ '"targetAddressSpace" in Request.prototype'
result: true
```

The page can also read the permission state before it calls the device:

```console
$ Q='Promise.all(["local-network", "loopback-network"].map(n => navigator.permissions.query({name: n}).then(s => n + "=" + s.state))).then(a => a.join(" | "))'
$ node cdp.mjs https://example.com/ "$Q"
result: local-network=prompt | loopback-network=prompt
$ node cdp.mjs https://example.com/ "$Q" local-network
granted local-network
result: local-network=granted | loopback-network=prompt
```

Clean up with the PIDs you saved, not by process name. Other local servers
may match the name:

```console
$ kill $CHROME_PID $LOOP $LAN
$ wait $CHROME_PID   # Chrome writes to the profile while it shuts down
$ cd /tmp && rm -rf /tmp/lna-mixed
```

## What to do about it

- **Calling an http LAN device from an https page:** pass
  `{ targetAddressSpace: "local" }` to `fetch`. Then make sure the user grants
  the local-network permission. Without the option, Chrome blocks the request as
  mixed content. In my test that happened even with the permission already
  granted, and the request never reached the device (`lan.log` stayed empty).
- **Name the address space the device is really in.** A wrong value fails with
  `Request had a target IP address space of X yet the resource is in address space Y`.
- **Calling `http://127.0.0.1` / `http://localhost`:** no option needed. You need
  `loopback-network`.
- **All of these look the same to page code:** `TypeError: Failed to fetch`.
  The reason is only in the console. Check `navigator.permissions.query({ name: "local-network" })`
  before you show the user an error.
- **If you control the device,** serving it over https avoids the mixed-content
  part completely. I didn't test that.

## Checklist

- `Mixed Content: ... This request has been blocked` and the device logs nothing
  → add `targetAddressSpace: "local"` (or `"loopback"`) to the fetch.
- After that you see `` Permission was denied ... `local` address space `` → the
  permission isn't granted yet.
- `Request had a target IP address space of local yet the resource is in address space loopback`
  → the value doesn't match where the host actually resolved.
- `http://127.0.0.1` and `http://localhost` are not mixed content. They only need
  `loopback-network`.
- A `Mixed Content ... should also be served over HTTPS` warning stays even
  when the request succeeds.

Not tested: a real LAN device or a real private IP (I used a `MAP` rule plus an
address-space override on `127.0.0.1`), `.local` mDNS names, the permission
prompt in a normal window, `XMLHttpRequest`, `WebSocket`, `<img>`/`<iframe>`
loads, a device served over https, Chrome versions other than 156, and other
browsers.

*Tested on macOS 26.5.2 (Apple silicon) with Google Chrome 156.0.8078.80
(`--headless=new`, throwaway profile in `/tmp`), Node v24.14.1 and the system
Python 3.8.2, in zsh 5.9. The page was `https://example.com/`. Both test servers
listened on 127.0.0.1 only.*
