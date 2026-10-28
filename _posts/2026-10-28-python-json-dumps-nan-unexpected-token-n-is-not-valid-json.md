---
title: "`SyntaxError: Unexpected token 'N', ... is not valid JSON`: Python `json.dumps` writes `NaN` and `Infinity` (and `jq`, `json.tool` don't complain)"
description: >-
  A Python endpoint returns `{"mean": NaN}` and the JavaScript side fails with
  "Unexpected token 'N' ... is not valid JSON" (`Infinity` gives
  "Unexpected token 'I'", `-Infinity` "No number after minus sign in
  JSON"). Python's `json.dumps` writes NaN, Infinity and
  -Infinity by default, `json.loads` and `python3 -m json.tool` accept them,
  and macOS `jq` 1.7.1 quietly turns them into null and ±1.797e308, so those
  three checks pass. `default=` was not called for a plain float here.
  Fix: `allow_nan=False` plus a function that replaces non-finite floats
  with None; regex-replacing NaN in JavaScript also rewrites strings.
tags: [python, json, javascript, node, jq, nan]
---

A small Python script writes stats as JSON for a web page. The JavaScript
that reads it fails with this (Node 24's message, from the repro below;
I didn't check the wording in browsers):

```console
SyntaxError: Unexpected token 'N', "{"mean": NaN, "wors"... is not valid JSON
```

The file opens fine in Python, `python3 -m json.tool` pretty-prints it, and
`jq` shows a value. The problem is one word in it: `NaN`. Python's `json`
module writes `NaN`, `Infinity` and `-Infinity` by default. Those aren't JSON,
and JavaScript's `JSON.parse` rejects them.

## Reproduce it

Two ordinary ways to get a non-finite float: `float()` accepts the string
`"NaN"` from a CSV cell without an error, and `max(..., default=-math.inf)`
returns `-inf` when the list is empty. Save as `stats.py`:

```python
# stats.py
import json, math

cells = ["12.5", "NaN", "8.0"]   # a CSV column; one cell says NaN
vals = [float(c) for c in cells]  # float("NaN") doesn't raise
errors = []                       # no errors in this run

print(json.dumps({
    "mean": sum(vals) / len(vals),
    "worst_error": max(errors, default=-math.inf),
}))
```

```console
$ python3 stats.py > stats.json
$ cat stats.json
{"mean": NaN, "worst_error": -Infinity}
```

Serve it and fetch it the way a page would, here with Node's `fetch` (Node
24, V8's `JSON.parse`):

```console
$ python3 -m http.server 8765 --bind 127.0.0.1 >/dev/null 2>&1 & pid=$!
[1] 72517
$ sleep 1
$ node -e 'fetch("http://127.0.0.1:8765/stats.json").then(r => r.json()).then(console.log, e => console.log(e.name + ": " + e.message))'
SyntaxError: Unexpected token 'N', "{"mean": NaN, "wors"... is not valid JSON
$ kill $pid
[1]  + terminated  python3 -m http.server 8765 --bind 127.0.0.1 > /dev/null 2>&1
```

The `[1] ...` job lines are what interactive zsh prints; the PID will
differ. `kill $pid` stops only the server started above (`$!` is its PID).

If the first bad value is an infinity, the message is different, so
searching for the `'N'` one doesn't find it. `Infinity` gives
`Unexpected token 'I'`, and `-Infinity` doesn't mention a token at all:

```console
$ node -e 'JSON.parse(String.raw`{"best": Infinity}`)' 2>&1 | grep SyntaxError
SyntaxError: Unexpected token 'I', "{"best": Infinity}" is not valid JSON
$ node -e 'JSON.parse(String.raw`{"worst_error": -Infinity}`)' 2>&1 | grep SyntaxError
SyntaxError: No number after minus sign in JSON at position 17 (line 1 column 18)
```

## Why the other checks pass

Python reads its own output back without complaint, so a Python round-trip
test passes:

```console
$ python3 -c 'import json; print(json.load(open("stats.json")))'
{'mean': nan, 'worst_error': -inf}
$ python3 -m json.tool stats.json; echo "exit $?"
{
    "mean": NaN,
    "worst_error": -Infinity
}
exit 0
```

`jq` on macOS (`/usr/bin/jq`, `jq-1.7.1-apple`) doesn't fail either. It turns
`NaN` into `null` and `-Infinity` into the most negative double, exit 0:

```console
$ jq . stats.json; echo "exit $?"
{
  "mean": null,
  "worst_error": -1.7976931348623157e+308
}
exit 0
```

So `curl ... | jq .` looks fine while `JSON.parse` fails. And going the other
way, JavaScript doesn't produce these: `JSON.stringify` writes `null` for
them.

```console
$ node -e 'console.log(JSON.stringify({a: NaN, b: -Infinity}))'
{"a":null,"b":null}
```

## `default=` doesn't help

A custom encoder looks like the place to fix this, but `default` is called
only for objects `json` can't serialize. A float is serializable, so
`default` never runs:

```console
$ python3 -c 'import json, math; print(json.dumps({"mean": math.nan}, default=lambda o: None))'
{"mean": NaN}
```

## Fix: fail on it, then replace it on purpose

First, `allow_nan=False` makes `json.dumps` raise instead of writing invalid
JSON:

```console
$ python3 -c 'import json, math; json.dumps({"mean": math.nan}, allow_nan=False)' 2>&1 | tail -n 1
ValueError: Out of range float values are not JSON compliant
```

That's Python 3.8.2. Newer Pythons may append the value; neither says which key it was.

Then decide what a missing number should be. Usually it's `null`. Walk the
data and replace non-finite floats before dumping. Save as `finite.py`:

```python
# finite.py
import json, math

def finite(o):
    """Copy of o with NaN/Infinity/-Infinity floats replaced by None."""
    if isinstance(o, float) and not math.isfinite(o):
        return None
    if isinstance(o, dict):
        return {k: finite(v) for k, v in o.items()}
    if isinstance(o, (list, tuple)):
        return [finite(v) for v in o]
    return o

def bad_paths(o, path="$"):
    """Where the non-finite floats are, for the log."""
    if isinstance(o, float) and not math.isfinite(o):
        yield path, o
    elif isinstance(o, dict):
        for k, v in o.items():
            yield from bad_paths(v, f"{path}.{k}")
    elif isinstance(o, (list, tuple)):
        for i, v in enumerate(o):
            yield from bad_paths(v, f"{path}[{i}]")

if __name__ == "__main__":
    data = {"mean": math.nan, "runs": [1.5, -math.inf], "note": "NaN = not measured"}
    for p, v in bad_paths(data):
        print("non-finite:", p, v)
    print(json.dumps(finite(data), allow_nan=False))
```

```console
$ python3 finite.py
non-finite: $.mean nan
non-finite: $.runs[1] -inf
{"mean": null, "runs": [1.5, null], "note": "NaN = not measured"}
```

The string `"NaN = not measured"` is left alone because only float values
are touched. Keep `allow_nan=False` on the final `dumps` even after
cleaning: if a new code path adds a NaN somewhere `finite` doesn't reach,
you get a `ValueError` on the server instead of a blank page.

## Don't patch it with a regex in JavaScript

Replacing `NaN` in the response text before `JSON.parse` "works" until a
string contains the word. `\b` doesn't save it:

```console
$ node -e 'console.log(JSON.parse(String.raw`{"note": "NaN = not measured", "avg": NaN}`.replace(/\bNaN\b/g, "null")))'
{ note: 'null = not measured', avg: null }
```

A `JSON.parse` reviver can't help either; the parse fails before the reviver
is called:

```console
$ node -e 'try { JSON.parse(String.raw`{"a": NaN}`, (k, v) => { console.log("reviver", k); return v }) } catch (e) { console.log(e.message) }'
Unexpected token 'N', "{"a": NaN}" is not valid JSON
```

Fix it where the JSON is written.

## Python can reject them on read, too

If Python reads JSON from somewhere else and you want the same rule as the
browser, `parse_constant` is called for `NaN`, `Infinity` and `-Infinity`:

```console
$ python3 -c '
import json
def no_constants(name):
    raise ValueError("not JSON: " + name)
json.loads("{\"mean\": NaN}", parse_constant=no_constants)
' 2>&1 | tail -n 1
ValueError: not JSON: NaN
```

## Checklist

- `Unexpected token 'N'` (or `'I'` for `Infinity`) "is not valid JSON", or
  `No number after minus sign in JSON` for `-Infinity`, from a Python
  backend: look for `NaN`, `Infinity`, `-Infinity` in the raw response.
- Python `json.dumps` writes them by default; `json.loads` and
  `python3 -m json.tool` accept them. A Python-only test won't catch it.
- macOS `jq` 1.7.1 prints them as `null` / `±1.7976931348623157e+308`, exit 0.
  Look at the raw body (`cat`, `curl` without `jq`).
- Common sources: `float("NaN")` from text data, `-math.inf` / `math.inf` as
  a starting or default value, `inf - inf` (not shown here).
- `default=` was not called for plain floats in my tests.
- Use `json.dumps(finite(data), allow_nan=False)`: replace non-finite
  floats with `None` on purpose, and fail loudly on any you missed.
- Don't regex-replace `NaN` in the JSON text; it changes strings too.

*Tested on 2026-10-28 on macOS 26.5.2 (build 25F84, Apple silicon) with
`/usr/bin/python3` 3.8.2 (plus Python 3.12.13 for the error message,
and the `default=` check), Node v24.14.1 (its built-in `fetch`), and
`/usr/bin/jq` jq-1.7.1-apple, in zsh 5.9 (`zsh -f`). Browsers (Chrome,
Safari, Firefox messages), other `jq` versions, Linux and libraries like
NumPy/pandas or `simplejson` were not tested.*
