---
title: "PWA shows a blank page offline after a deploy: the service worker never cached the new hashed bundle"
description: >-
  A small PWA worked offline until a deploy. Opened once online after the
  deploy and then offline, it gave an empty page and "net::ERR_FAILED" for
  /assets/app.<hash>.js.
  The service worker precached "/" at install but left the JavaScript to be
  cached on first use, and the new HTML points at a bundle name the browser has
  never fetched. Reproduced with headless Chromium (Playwright), simulating
  offline by stopping the local server, then fixed by precaching the hashed
  bundles at install.
tags: [javascript, pwa, service-worker, web]
---

Your PWA works offline. You deploy a new version, open the app once while
online (it still shows the old version, which is normal), and later open it
offline: **the page is blank**. The HTML comes from the service worker, but
the request for `/assets/app.<hash>.js` fails with `net::ERR_FAILED`.
In my test, offline worked once the page had been reloaded online at least
once (the "day one" step below), because the reload caches the bundle.

I reproduced this on my Mac with headless Chromium. "Offline" in this post
means I stopped the local server, so only the service worker's caches could
answer; I didn't test with the network actually cut (Wi-Fi off, airplane
mode). The service worker cached `/` at install and cached everything else
"on the way through" the first time it was requested. After a deploy the new
service worker caches the **new** `index.html`, which points at a **new**
hashed bundle name. Nothing has requested that file yet, so it isn't in any
cache, and the activate step has just deleted the old cache that held the old
bundle. The cached page refers to a script that only exists on the server.

## Reproduce it

Four small files in an empty folder. `build.py` plays the role of a deploy: it
replaces `site/` with an `index.html`, one content-hashed bundle and `sw.js`.
With `--precache` the bundle goes into the service worker's precache list;
without it only `/` does.

```python
# build.py — "deploy" a tiny app into site/: python3 build.py v1 [--precache]
import hashlib, json, pathlib, shutil, sys

version = sys.argv[1]
precache_bundles = "--precache" in sys.argv
site = pathlib.Path("site")
shutil.rmtree(site, ignore_errors=True)        # a deploy replaces the old files
(site / "assets").mkdir(parents=True)

js = f'document.getElementById("root").textContent = "notes app {version}";\n'
name = f"app.{hashlib.sha256(js.encode()).hexdigest()[:8]}.js"
(site / "assets" / name).write_text(js)
bundles = [f"/assets/{name}"]

(site / "index.html").write_text(f"""<!doctype html>
<title>notes</title>
<div id="root"></div>
<script src="/assets/{name}"></script>
<script>navigator.serviceWorker.register("/sw.js");</script>
""")

precache = ["/"] + (bundles if precache_bundles else [])
sw = pathlib.Path("sw.template.js").read_text()
sw = sw.replace("__VERSION__", version).replace("__PRECACHE__", json.dumps(precache))
(site / "sw.js").write_text(sw)
print(version, name, "precache:", precache)
```

The service worker is a common cache-first pattern: precache a list at install,
delete other versions' caches at activate, and cache anything else the first
time it's fetched:

```js
// sw.template.js
const CACHE = "app-__VERSION__";
const PRECACHE = __PRECACHE__;

self.addEventListener("install", (e) => {
  e.waitUntil(
    caches.open(CACHE)
      .then((c) => c.addAll(PRECACHE))
      .then(() => self.skipWaiting())
  );
});

self.addEventListener("activate", (e) => {
  e.waitUntil(
    caches.keys()
      .then((keys) => Promise.all(keys.filter((k) => k !== CACHE).map((k) => caches.delete(k))))
      .then(() => self.clients.claim())
  );
});

// cache first; anything not cached yet is fetched and cached on the way through
self.addEventListener("fetch", (e) => {
  if (e.request.method !== "GET") return;
  e.respondWith(
    caches.match(e.request).then((hit) => hit || fetch(e.request).then((res) => {
      if (res.ok) {
        const copy = res.clone();
        caches.open(CACHE).then((c) => c.put(e.request, copy));
      }
      return res;
    }))
  );
});
```

A static server on `127.0.0.1` only (service workers are allowed on
`http://127.0.0.1` without HTTPS). It sends `Cache-Control: no-cache` so the
browser's HTTP cache doesn't hide the effect:

```python
# server.py — serve site/ on 127.0.0.1:8771, always revalidate
import functools
from http.server import ThreadingHTTPServer, SimpleHTTPRequestHandler

class Handler(SimpleHTTPRequestHandler):
    def end_headers(self):
        self.send_header("Cache-Control", "no-cache")
        super().end_headers()
    def log_message(self, *args):
        pass

handler = functools.partial(Handler, directory="site")
ThreadingHTTPServer(("127.0.0.1", 8771), handler).serve_forever()
```

And a Playwright script that opens the app in a **persistent** headless
Chromium profile (`profile/`), so the service worker and its caches survive
from one run to the next, like a real browser. It prints what's in `#root`
and what's in each cache. "Offline" here means the server is stopped, so
nothing answers on `127.0.0.1:8771` and only the service worker's caches can
serve the page (why not Playwright's offline mode: see below).

```js
// visit.mjs — open the app in a persistent headless Chromium profile
// node visit.mjs            online: load once, wait for the newest service worker
// node visit.mjs --reload   same, then reload once
// node visit.mjs --offline  one load, no waiting (stop the server first)
import { chromium } from "playwright";

const offline = process.argv.includes("--offline");
const ctx = await chromium.launchPersistentContext("profile", { headless: true });
const page = ctx.pages()[0] ?? await ctx.newPage();
page.on("requestfailed", (r) => console.log("  failed:", new URL(r.url()).pathname, r.failure().errorText));

const show = async (label) => {
  const root = await page.textContent("#root").catch(() => "(no page)");
  const keys = await page.evaluate(async () => {
    const out = [];
    for (const k of await caches.keys()) {
      const reqs = await (await caches.open(k)).keys();
      out.push(`${k}: ${reqs.map((r) => new URL(r.url).pathname).join(" ")}`);
    }
    return out;
  }).catch(() => []);
  console.log(`${label}: #root = ${JSON.stringify(root)}`);
  for (const k of keys) console.log("  cache", k);
};

if (offline) {
  await page.goto("http://127.0.0.1:8771/").catch((e) => console.log("  goto:", e.message.split("\n")[0]));
  await page.waitForTimeout(500);
  await show("offline");
} else {
  await page.goto("http://127.0.0.1:8771/");
  // wait until the newest sw.js is installed and controlling the page
  await page.evaluate(async () => {
    const reg = await navigator.serviceWorker.ready;
    await reg.update();
    while (reg.installing || reg.waiting || !navigator.serviceWorker.controller) {
      await new Promise((r) => setTimeout(r, 100));
    }
    // give activate up to 3 s to delete the previous version's cache
    for (let i = 0; i < 30 && (await caches.keys()).length > 1; i++) {
      await new Promise((r) => setTimeout(r, 100));
    }
  });
  await page.waitForTimeout(500);
  await show("online");
  if (process.argv.includes("--reload")) {
    await page.reload();
    await page.waitForTimeout(500);
    await show("online, reload");
  }
}
await ctx.close();
```

Install Playwright (`npx playwright install chromium` downloads the browser
if you don't have it; I already had it), then start the server and keep its
PID — you'll stop and start it to go "offline" and back:

```console
$ npm init -y >/dev/null && npm i playwright && npx playwright install chromium
$ python3 server.py & SERVER=$!
```

Day one: deploy v1, open the app and reload once (so the bundle gets cached on
the way through), then stop the server and open it again. Everything works:

```console
$ python3 build.py v1 && node visit.mjs --reload
v1 app.bbd8455b.js precache: ['/']
online: #root = "notes app v1"
  cache app-v1: /
online, reload: #root = "notes app v1"
  cache app-v1: / /assets/app.bbd8455b.js
$ kill $SERVER
$ node visit.mjs --offline
offline: #root = "notes app v1"
  cache app-v1: / /assets/app.bbd8455b.js
```

Deploy v2, open the app **once** while online, then go offline:

```console
$ python3 server.py & SERVER=$!
$ python3 build.py v2 && node visit.mjs
v2 app.05452931.js precache: ['/']
online: #root = "notes app v1"
  cache app-v2: /
$ kill $SERVER
$ node visit.mjs --offline
  failed: /assets/app.05452931.js net::ERR_FAILED
offline: #root = ""
  cache app-v2: /
```

That's the bug. The online visit still showed v1 (the page was served from the
old cache before the new worker took over). The new worker cached the v2
`index.html`, and activate deleted `app-v1` with the only bundle in it. Offline,
the v2 HTML asks for `app.05452931.js`, which no cache has, and `#root` stays
empty. In a real app that's a white screen. I ran the same deploy sequence
again from a fresh profile: blank in 5 of 5 runs.

It's not only deploys. On a **first visit** the bundle loads before the service
worker exists, so it isn't cached either. With a fresh profile, opening v1 once
and then going offline was blank too (one run):

```console
$ rm -rf profile
$ python3 server.py & SERVER=$!
$ python3 build.py v1 && node visit.mjs
v1 app.bbd8455b.js precache: ['/']
online: #root = "notes app v1"
  cache app-v1: /
$ kill $SERVER
$ node visit.mjs --offline
  failed: /assets/app.bbd8455b.js net::ERR_FAILED
offline: #root = ""
  cache app-v1: /
```

### Why I stopped the server instead of using `setOffline`

My first version of `visit.mjs` used Playwright's `context.setOffline(true)`.
In 1 of 6 runs of the deploy sequence, the offline load **worked** and the
v2 bundle showed up in the cache afterwards. I didn't find out why; my guess
is that the emulation didn't block the service worker's own `fetch()` that
time, but I haven't confirmed it. A test that passes by luck hides exactly
this bug, so I switched to stopping the server, which gave a blank page in
7 of 7 runs of the deploy sequence (the 5 above plus two full runs of this
post). If you test offline behaviour with network emulation, check that the
result is repeatable.

## Fix: put the hashed bundles in the precache list

With a cache-first worker like this one, the fix is to cache, at install,
the files the cached HTML will ask for, so the two arrive together. The file
names change on each build, so the list has to come from the build: here
`build.py --precache` writes the bundle names into `sw.js`. Same
steps as above, starting from a fresh profile:

```console
$ rm -rf profile
$ python3 server.py & SERVER=$!
$ python3 build.py v1 --precache && node visit.mjs
v1 app.bbd8455b.js precache: ['/', '/assets/app.bbd8455b.js']
online: #root = "notes app v1"
  cache app-v1: / /assets/app.bbd8455b.js
$ kill $SERVER
$ node visit.mjs --offline
offline: #root = "notes app v1"
  cache app-v1: / /assets/app.bbd8455b.js
$ python3 server.py & SERVER=$!
$ python3 build.py v2 --precache && node visit.mjs
v2 app.05452931.js precache: ['/', '/assets/app.05452931.js']
online: #root = "notes app v1"
  cache app-v2: / /assets/app.05452931.js
$ kill $SERVER
$ node visit.mjs --offline
offline: #root = "notes app v2"
  cache app-v2: / /assets/app.05452931.js
```

The first visit now works offline, and after the deploy the offline page is v2
with its bundle — the HTML and the scripts it names are installed together.

### If a listed file is missing, the update doesn't happen

`cache.addAll()` is all-or-nothing: if any URL fails, the promise rejects and
the install fails. I checked by deploying v3 with `--precache` and then
deleting its bundle from `site/` before visiting:

```console
$ python3 server.py & SERVER=$!
$ python3 build.py v3 --precache && rm site/assets/app.*.js && node visit.mjs
v3 app.7b8eea49.js precache: ['/', '/assets/app.7b8eea49.js']
online: #root = "notes app v2"
  cache app-v2: / /assets/app.05452931.js
  cache app-v3: 
$ kill $SERVER
$ node visit.mjs --offline
offline: #root = "notes app v2"
  cache app-v2: / /assets/app.05452931.js
  cache app-v3: 
```

The v3 worker never activated: the v2 worker stayed in control and the app
still worked offline as v2. The failed install did leave an empty `app-v3`
cache behind (`caches.open()` creates it before `addAll` fails). The activate
step in `sw.template.js` deletes every cache except its own, so the next
successful version should clean it up; I didn't run that step. Keeping users
on the last working version is what you want from a broken deploy, instead of
installing an HTML page that points at nothing.

## Checklist

- With a cache-first worker like this one that precaches only the HTML and
  leaves the bundles to runtime caching, **the first offline open after a
  deploy can be blank** if the user opened the new version only once online
  (and the same after a single first visit). Putting the scripts the HTML
  references into the install-time precache fixed both in my test. A later
  online reload caches the bundle too, so not every user hits it.
- Generate the list from the build output (the hashed names change every
  build). Here a script writes it into `sw.js`. Bundler plugins for Workbox or
  Vite's `build.manifest` can produce such a list; I didn't test those here.
- Use `cache.addAll()` for the list so a missing file fails the install and
  the old version stays, instead of installing half a version.
- Test it the way users hit it: deploy, open **once** online (no reload), then
  go offline. Testing offline right after a reload hides the bug, because the
  reload caches the bundle on the way through.
- If you test with network emulation (Playwright's `setOffline`), repeat the
  run: in 1 of my 6 runs the offline load worked when it shouldn't have.
  Stopping the server gave the same result in 7 of 7 runs here.
- When you test locally, make sure the HTTP cache isn't involved
  (`Cache-Control: no-cache` here). Otherwise `addAll` may be served a stale
  `index.html` from the HTTP cache; I set the header so I didn't have to find
  out.
- Stop the test server with `kill` and the PID it was started with, not by
  process name.

*Tested on macOS 26.5.2 (Apple silicon) with Python 3.8.2 (`/usr/bin/python3`),
Node 24.14.1 and Playwright 1.62.1 driving its bundled headless Chromium
(Chrome for Testing 151). Server bound to `127.0.0.1` only. I didn't test
Safari/WebKit, Firefox, or an installed (home-screen) PWA; I'd expect the same
result from the same caching logic, but I haven't run it there.*
