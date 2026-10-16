---
title: "Text jumps back while typing: the server's change event echoes your own save and the refetch overwrites it (SSE)"
description: >-
  An autosaving editor that refetches the document on every server "changed"
  event kept eating characters: the text jumped back mid-word and the saved
  result was garbled. The change event for your own save arrives after you've
  typed more, and the refetch returns older text. Reproduced with a small Python
  SSE server and a Node client, then fixed with a per-session write sequence
  number. Skipping events with your own session id is not enough.
tags: [javascript, python, sse, web]
---

You type into an editor that autosaves. Every so often the text jumps back a few
characters, the next keys land in the wrong place, and what ends up saved is
garbled — `the quick brown fox jumps over the lazy dog` came out as
`theicox jse lg` in the repro below. Nothing errors, and every save reaches
the server.

The editor listens for "document changed" events from the server (Server-Sent
Events here) so that edits from another tab or device show up. When an event
arrives it refetches the document and puts the result in the editor. The catch:
**the server sends that event for your own save too**. By the time the event
arrives and the refetch comes back, you've typed more, so the refetch hands you
*older* text and the editor rolls back to it. Your next keystroke then saves the
rolled-back text, and the server stores it.

I reproduced this on my Mac with a small server and a scripted typist, then
fixed it with a per-session write sequence number.

## Reproduce it

A server with one shared document. `PUT /doc` saves it, `GET /doc` returns it,
and `GET /events` is an SSE stream with one event per save. The save sleeps for a
moment to stand in for a slow disk, network or sync step:

```python
# server.py — one shared document, saved with PUT, change events over SSE
import json, queue, sys, threading, time
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler

SAVE_DELAY = float(sys.argv[1]) if len(sys.argv) > 1 else 0.08   # seconds a save takes
doc = {"text": "", "seqs": {}}   # seqs: last write applied per session
lock = threading.Lock()
listeners = []

class Handler(BaseHTTPRequestHandler):
    def log_message(self, *args):
        pass

    def do_GET(self):
        if self.path == "/events":
            q = queue.Queue()
            with lock:
                listeners.append(q)
            self.send_response(200)
            self.send_header("Content-Type", "text/event-stream")
            self.end_headers()
            try:
                while True:
                    self.wfile.write(b"data: " + q.get().encode() + b"\n\n")
                    self.wfile.flush()
            except OSError:
                with lock:
                    listeners.remove(q)
            return
        with lock:
            body = json.dumps(doc).encode()
        self.reply(body)

    def do_PUT(self):
        msg = json.loads(self.rfile.read(int(self.headers["Content-Length"])))
        time.sleep(SAVE_DELAY)
        with lock:
            if msg["seq"] > doc["seqs"].get(msg["session"], 0):   # drop late, older writes
                doc["text"] = msg["text"]
                doc["seqs"][msg["session"]] = msg["seq"]
            event = json.dumps({"session": msg["session"], "seq": msg["seq"]})
            for q in listeners:
                q.put(event)
        self.reply(b"{}")

    def reply(self, body):
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

ThreadingHTTPServer(("127.0.0.1", 8767), Handler).serve_forever()
```

Every save carries a random `session` id (one per open editor) and a `seq` that
counts that session's saves. The server already uses them to drop a save that
arrives after a newer one from the same session, so out-of-order requests can't
store old text — that is *not* the bug here. It also records the last applied
`seq` per session in `doc.seqs`, which the fix below uses.

The client is a Node script that behaves like the editor: it presses one key
every 30 ms, saves after every key, and refetches on every change event. With no
flag it applies whatever the refetch returns. It also measures how long each of
its own saves took to come back as an event:

```js
// editor.mjs — types a sentence, saves after every keystroke,
// and refetches the document whenever the server says it changed.
//   node editor.mjs             apply every refetch (the bug)
//   node editor.mjs --skip-own  ignore events for my own saves
//   node editor.mjs --fix       ignore refetches that don't include all my saves
const BASE = "http://127.0.0.1:8767";
const MODE = process.argv[2] ?? "";
const session = Math.random().toString(36).slice(2);
const sentence = "the quick brown fox jumps over the lazy dog";
const KEY_MS = Number(process.env.KEY_MS ?? 30);   // time between key presses
let text = "";        // what the user sees in the editor
let seq = 0;          // saves this session has sent
let overwrites = 0;   // times a refetch replaced what the editor showed
const sentAt = {};    // seq -> when that save was sent
const echoMs = [];    // ms from sending a save to its event arriving

function save() {
  seq += 1;
  sentAt[seq] = performance.now();
  fetch(`${BASE}/doc`, {
    method: "PUT",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ session, seq, text }),
  });
}

async function onChange(event) {
  if (event.session === session) echoMs.push(performance.now() - sentAt[event.seq]);
  if (MODE === "--skip-own" && event.session === session) return;
  const doc = await (await fetch(`${BASE}/doc`)).json();
  if (MODE === "--fix" && (doc.seqs[session] ?? 0) < seq) return;  // server hasn't got all my saves yet
  if (doc.text !== text) {
    overwrites += 1;
    console.log(`event ${event.session === session ? "mine" : event.session} seq=${event.seq}: editor "${text}" -> "${doc.text}"`);
    text = doc.text;
  }
}

// listen for change events (Server-Sent Events)
const res = await fetch(`${BASE}/events`);
const decoder = new TextDecoder();
(async () => {
  let buf = "";
  for await (const chunk of res.body) {
    buf += decoder.decode(chunk, { stream: true });
    let i;
    while ((i = buf.indexOf("\n\n")) >= 0) {
      const line = buf.slice(0, i);
      buf = buf.slice(i + 2);
      onChange(JSON.parse(line.slice("data: ".length)));
    }
  }
})();

// the user presses one key every KEY_MS; each key goes at the end of what the editor shows
for (const key of sentence) {
  text += key;
  save();
  await new Promise((r) => setTimeout(r, KEY_MS));
}
await new Promise((r) => setTimeout(r, 1000));  // let the last events arrive
const ms = echoMs.sort((a, b) => a - b).map(Math.round);
console.log(`final editor: "${text}"`);
console.log(`keys pressed: ${sentence.length}, overwrites: ${overwrites}, ` +
  `save->event ms: min ${ms[0]} median ${ms[ms.length >> 1]} max ${ms.at(-1)}`);
process.exit(0);
```

Start the server, keep its PID, and type (output trimmed in the middle):

```console
$ python3 server.py & SERVER=$!
$ node editor.mjs
event mine seq=3: editor "the qu" -> "the"
event mine seq=5: editor "theic" -> "the q"
event mine seq=4: editor "the qk " -> "theic"
event mine seq=6: editor "theicbr" -> "theic"
event mine seq=9: editor "theicow" -> "the qk"
...
event mine seq=42: editor "theicox jsee d" -> "theicox jsee do"
event mine seq=43: editor "theicox jsee do" -> "theicox jse lg"
final editor: "theicox jse lg"
keys pressed: 43, overwrites: 31, save->event ms: min 88 median 203 max 233
```

The first line is the bug in one step. The editor shows `the qu`; the event for
save 3 arrives; the refetch returns `the`, because that's all the server had
stored at that moment; the editor drops back to `the`. The next keys are added to
the rolled-back text (`theic` — the `qu` is gone), those saves go out, their
echoes roll the editor back again, and it snowballs. Events also arrive out of
order (5 before 4): the server handles saves on separate threads, so they don't
finish in the order they were sent.

And it isn't only on screen — the garbled text is what got saved:

```console
$ curl -s http://127.0.0.1:8767/doc
{"text": "theicox jse lg", "seqs": {"lpd5f2hm3hh": 43}}
```

Earlier runs the same morning: one ended as `the qubx oe  dog` (25 overwrites),
and three more gave 34, 26 and 28 overwrites out of 43 keys.

## It depends on typing speed vs. the save round trip

Slower typing helps only once the gap between keys is longer than *save + event +
refetch*. Three runs at each key interval, against the same server:

```console
$ for k in 120 200 300 500; do for i in 1 2 3; do KEY_MS=$k node editor.mjs | tail -1; done; done
keys pressed: 43, overwrites: 41, save->event ms: min 84 median 222 max 241
keys pressed: 43, overwrites: 41, save->event ms: min 106 median 224 max 235
keys pressed: 43, overwrites: 43, save->event ms: min 124 median 219 max 238
keys pressed: 43, overwrites: 43, save->event ms: min 124 median 224 max 235
keys pressed: 43, overwrites: 36, save->event ms: min 96 median 210 max 234
keys pressed: 43, overwrites: 39, save->event ms: min 104 median 223 max 262
keys pressed: 43, overwrites: 0, save->event ms: min 86 median 200 max 241
keys pressed: 43, overwrites: 0, save->event ms: min 93 median 217 max 236
keys pressed: 43, overwrites: 0, save->event ms: min 91 median 214 max 235
keys pressed: 43, overwrites: 0, save->event ms: min 107 median 219 max 242
keys pressed: 43, overwrites: 0, save->event ms: min 87 median 224 max 237
keys pressed: 43, overwrites: 0, save->event ms: min 104 median 213 max 242
```

At 120 and 200 ms between keys almost every event rolled the editor back; at 300
and 500 ms none did. The last column says why: a save took about 85–260 ms
(median a bit over 200) to come back as an event, so at 200 ms the next key was
usually pressed before the refetch returned, and at 300 ms it never was.

That's longer than the 0.08 s `SAVE_DELAY`. Part of it is `time.sleep` itself:
in the session I ran this from, five calls to `time.sleep(0.08)` timed with
`time.perf_counter()` took 0.12–0.23 s. I didn't dig into why. The measured
save-to-event time is the number that matters, whatever its cause.

Make the save slower and the cut-off moves with it (I've left out the shell's
job messages):

```console
$ kill $SERVER; python3 server.py 0.3 & SERVER=$!
$ for k in 250 300 500 700; do KEY_MS=$k node editor.mjs | tail -1; done
keys pressed: 43, overwrites: 43, save->event ms: min 309 median 443 max 457
keys pressed: 43, overwrites: 43, save->event ms: min 305 median 433 max 463
keys pressed: 43, overwrites: 0, save->event ms: min 307 median 430 max 455
keys pressed: 43, overwrites: 0, save->event ms: min 314 median 446 max 462
```

And with no delay at all, it never showed up:

```console
$ kill $SERVER; python3 server.py 0 & SERVER=$!
$ for i in 1 2 3; do node editor.mjs | tail -1; done
keys pressed: 43, overwrites: 0, save->event ms: min 1 median 3 max 6
keys pressed: 43, overwrites: 0, save->event ms: min 1 median 3 max 7
keys pressed: 43, overwrites: 0, save->event ms: min 1 median 4 max 6
```

So a user who pauses longer than your save round trip won't see it, and one who
types faster will. On localhost with a tiny document the round trip is a few
milliseconds, which is why this can pass local testing and break once there's a
slow disk, a sync step or a real network in the save path.

## Fix: a per-session write sequence

The client already numbers its saves (`seq`), and the server records the last
`seq` it applied for each session (`doc.seqs`) and returns it with the document.
So the client can tell whether a refetched document **includes all of its own
saves**. If it doesn't, the editor is ahead of the server and the refetch must
not overwrite it:

```js
if (MODE === "--fix" && (doc.seqs[session] ?? 0) < seq) return;  // server hasn't got all my saves yet
```

`seq` is read when the refetch *returns*, not when the event arrived. If you
typed (and saved) while the refetch was in flight, the check sees the new `seq`
and skips that stale response too.

With the check, the same typing at 30 ms per key gave no overwrites, and neither
did the slow server at 250 ms per key, which had overwritten all 43 keys before:

```console
$ kill $SERVER; python3 server.py & SERVER=$!
$ for i in 1 2 3; do node editor.mjs --fix | tail -1; done
keys pressed: 43, overwrites: 0, save->event ms: min 96 median 223 max 233
keys pressed: 43, overwrites: 0, save->event ms: min 91 median 209 max 234
keys pressed: 43, overwrites: 0, save->event ms: min 97 median 203 max 235
$ kill $SERVER; python3 server.py 0.3 & SERVER=$!
$ KEY_MS=250 node editor.mjs --fix | tail -1
keys pressed: 43, overwrites: 0, save->event ms: min 308 median 444 max 456
```

Each fixed run ended with the full sentence in the editor, and on the server
(checked with `GET /doc` after a repeat of the slow-server run).

## Why not just skip events with my own session id?

That's the obvious first fix (`--skip-own` in the script), and on its own it
removes the echo:

```console
$ kill $SERVER; python3 server.py & SERVER=$!
$ for i in 1 2 3; do node editor.mjs --skip-own | tail -1; done
keys pressed: 43, overwrites: 0, save->event ms: min 95 median 212 max 235
keys pressed: 43, overwrites: 0, save->event ms: min 93 median 209 max 236
keys pressed: 43, overwrites: 0, save->event ms: min 85 median 208 max 234
```

The difference shows up when **another** tab saves while you type. Its event is
not yours, so the editor refetches — and that refetch can return a document
holding an *older copy of your own text*, because your latest saves are still in
flight. To test it, a second "tab" (`curl`, session `tab-b`) saves four times,
about half a second apart, while the editor types:

```console
$ kill $SERVER; python3 server.py & SERVER=$!
$ node editor.mjs --skip-own & sleep 0.25; for n in 1 2 3 4; do curl -s -o /dev/null -X PUT \
    -H "Content-Type: application/json" -d '{"session":"tab-b","seq":'$n',"text":"B'$n' "}' \
    http://127.0.0.1:8767/doc; sleep 0.25; done; wait $!
event tab-b seq=1: editor "the quick b" -> "B1 "
event tab-b seq=2: editor "B1 rown fox jumps over " -> "B1 rown fox jumps"
event tab-b seq=3: editor "B1 rown fox jumpsthe lazy dog" -> "B3 "
event tab-b seq=4: editor "B3 " -> "B4 "
final editor: "B4 "
keys pressed: 43, overwrites: 4, save->event ms: min 87 median 208 max 275
```

Look at the second line. The event is tab-b's, but the refetch returned
`B1 rown fox jumps` — this editor's own text from a few keys earlier, because
tab-b's save 2 had been overwritten on the server by this editor's later saves,
and its newest saves hadn't landed yet. The editor rolled back to it and the
next keys ran together (`jumpsthe`). That happened in 3 of 6 runs (restarting the
server before each so `tab-b` starts at `seq` 1 again); in the other three, every
refetch happened to return tab-b's text.

Six runs of the same test with `--fix` never rolled back to the editor's own
text. The tab-b saves whose refetch came back while the editor still had saves in
flight were skipped (`B1` and `B2` in every run, `B3` too in one), and the later
ones were shown:

```console
$ kill $SERVER; python3 server.py & SERVER=$!
$ node editor.mjs --fix & sleep 0.25; for n in 1 2 3 4; do curl -s -o /dev/null -X PUT \
    -H "Content-Type: application/json" -d '{"session":"tab-b","seq":'$n',"text":"B'$n' "}' \
    http://127.0.0.1:8767/doc; sleep 0.25; done; wait $!
event tab-b seq=3: editor "the quick brown fox jumps over the lazy dog" -> "B3 "
event tab-b seq=4: editor "B3 " -> "B4 "
final editor: "B4 "
keys pressed: 43, overwrites: 2, save->event ms: min 86 median 217 max 235
```

That's not a merge either: tab-b's `B1` and `B2` were stored and then replaced
by this editor's next save, and the user never saw them. Whose text should win
when two people type into the same field at once is a different problem
(locking, operational transforms, CRDTs). What the sequence check fixes is
narrower and is the bug in the title: **your own edits aren't undone by a stale
copy of themselves**, whichever event triggered the refetch.

When you're done, stop the server with the PID you saved: `kill $SERVER`.

## Checklist

- Text jumps back while typing, or saved text is missing characters, in an
  editor that refetches on server change events: suspect the echo of your own save.
- It may not show up on localhost with a fast save. Add a delay to the save path
  and type faster than the save round trip.
- Give each editor instance a session id and number its saves; have the server
  record the last applied number per session and return it with the document.
- Before applying a refetched document, check that it includes all your saves
  (`doc.seqs[session] >= seq`), reading `seq` when the response arrives.
- Have the server drop a save whose number is lower than the last one it applied
  for that session, so out-of-order requests can't store old text.
- Skipping your own session's events is not enough: a refetch triggered by
  someone else's change can still carry your own stale text.
- This doesn't merge concurrent edits. If two people really type into the same
  text, you need locking, operational transforms or CRDTs.
- If you debounce saves, unsaved local edits also make the editor newer than any
  refetch. I only tested saving on every key; with a debounce you'd also skip
  refetches while there are unsaved edits.

*Tested on macOS 26.5.2 (Apple silicon) with Python 3.8.2 (`/usr/bin/python3`,
Command Line Tools), Node 24.14.1 (built-in `fetch`) and curl 8.7.1. Server bound
to `127.0.0.1` only. I didn't test in a browser (`EventSource`) or over
WebSocket; the client logic would be the same, but I haven't run it there.*
