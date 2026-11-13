---
title: "macOS `base64 -d` cuts off a JWT payload: jq \"Unfinished JSON term at EOF\" (exit 0)"
description: >-
  Decoding the middle part of a JWT with `cut -d. -f2 | base64 -d` on a Mac
  prints JSON with the last one or two bytes missing — `"exp":176722560`
  instead of `"exp":1767225600`, no closing brace — and base64 exits 0. jq
  then fails with "Unfinished JSON term at EOF". JWTs drop the `=` padding,
  and /usr/bin/base64 on macOS throws away the last incomplete group of
  characters without a warning. Add the padding back (length % 4 == 2 → "==",
  3 → "="), or use jq's @base64d or Python with the padding added.
tags: [macos, base64, jwt, jq, shell]
---

The usual one-liner for looking inside a JWT, run on a Mac:

```console
$ TOKEN='eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsInJvbGUiOiJhZG1pbiIsImV4cCI6MTc2NzIyNTYwMH0.c2lnbmF0dXJlLW5vdC1jaGVja2Vk'
$ echo "$TOKEN" | cut -d. -f2 | base64 -d | jq .
jq: parse error: Unfinished JSON term at EOF at line 1, column 42
```

(This is a made-up token for the examples: header `{"alg":"HS256","typ":"JWT"}`,
and the "signature" is just the text `signature-not-checked`.)

Without jq you can see why it fails:

```console
$ echo "$TOKEN" | cut -d. -f2 | base64 -d; echo
{"sub":"42","role":"admin","exp":176722560
$ echo "$TOKEN" | cut -d. -f2 | base64 -d > /dev/null; echo "exit $?"
exit 0
```

The JSON stops in the middle of a number. The closing `}` is gone, and so is
the last digit of `exp`. `base64` didn't print an error and exited 0.

jq at least refuses the broken JSON. A script that pulls the field out with
`grep` does not, and gets a different number:

```console
$ echo "$TOKEN" | cut -d. -f2 | base64 -d | grep -o '"exp":[0-9]*'
"exp":176722560
$ date -u -r 176722560
Fri Aug  8 09:36:00 UTC 1975
$ date -u -r 1767225600
Thu Jan  1 00:00:00 UTC 2026
```

The token expires in 2026. The cut-off value says 1975, so an "is it expired?"
check would say yes.

## Why: JWTs drop the `=` padding

Base64 turns every 3 bytes into 4 characters. When the input length isn't a
multiple of 3, the last group is short, and standard Base64 fills it up to 4
characters with `=`. JWTs use the "base64url" form, which leaves the `=` off.
So the middle part of a token is often 2 or 3 characters short of a multiple
of 4:

```console
$ p=$(echo "$TOKEN" | cut -d. -f2); echo "${#p} characters, ${#p} % 4 = $(( ${#p} % 4 ))"
59 characters, 59 % 4 = 3
$ printf '%s' "$p" | base64 -d | wc -c
      42
$ printf '%s=' "$p" | base64 -d | wc -c
      44
$ printf '%s=' "$p" | base64 -d | jq -c .
{"sub":"42","role":"admin","exp":1767225600}
```

Without the `=`, `/usr/bin/base64` on this Mac drops the incomplete last group
(3 characters, which hold 2 bytes) and exits 0. With one `=` added, all 44
bytes come back.

With a short string it is easier to see. `aGk=` is `hi`; without the `=`
nothing at all comes out:

```console
$ printf 'aGk' | base64 -d | wc -c
       0
$ printf 'aGk=' | base64 -d; echo
hi
```

How much goes missing depends on the payload length: 1 byte when 2 characters
are left over, 2 bytes when 3 are. A payload whose length happens to be a
multiple of 4 decodes fine — which is why the same one-liner works for some
tokens and not for others.

Other decoders on the same Mac:

```console
$ python3 -c 'import base64,sys; base64.urlsafe_b64decode(sys.argv[1])' "$p" 2>&1 | tail -n 1
binascii.Error: Incorrect padding
$ printf '%s' "$p" | openssl base64 -d -A | wc -c
      42
```

Python refuses the input ("Incorrect padding") instead of guessing.
`/usr/bin/openssl` (LibreSSL 3.3.6) cut it off the same way as `base64`. A
Homebrew OpenSSL 3.6.3 I also tried printed nothing at all for this input,
again with exit 0.

base64url also uses `-` and `_` where standard Base64 uses `+` and `/`. The
macOS `base64` decodes both forms the same way, so that part isn't what broke
here:

```console
$ printf '%s' '-_-_' | base64 -d | xxd -p
fbffbf
$ printf '%s' '+/+/' | base64 -d | xxd -p
fbffbf
```

But jq's `@base64d` (below) doesn't accept `-` and `_`, and other `base64`
programs may not either, so the fix changes them anyway.

## Fix 1: add the padding back (shell)

```sh
#!/bin/sh
# Print the payload (middle part) of a JWT as text. Does NOT check the signature.
b64url_decode() {
  s=$(printf '%s' "$1" | tr '_-' '/+')
  case $(( ${#s} % 4 )) in
    0) ;;
    2) s="$s==" ;;
    3) s="$s=" ;;
    *) echo "b64url_decode: impossible length ${#s}" >&2; return 1 ;;
  esac
  printf '%s' "$s" | base64 -d
}

b64url_decode "$(printf '%s' "$1" | cut -d. -f2)"
```

Save it as `jwtpayload.sh` (I used an empty folder from `mktemp -d`) and
`chmod +x jwtpayload.sh`. A remainder of 1 can't
come from real Base64 (the token was cut or copied wrong), so the function
stops with exit 1 instead of quietly returning less.

```console
$ ./jwtpayload.sh "$TOKEN" | jq -c .
{"sub":"42","role":"admin","exp":1767225600}
$ ./jwtpayload.sh "$TOKEN" | jq .exp
1767225600
$ T2='eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsIm5hbWUiOiI8PD4-PyIsImV4cCI6MTc2NzIyNTYwMH0.c2lnbmF0dXJlLW5vdC1jaGVja2Vk'
$ ./jwtpayload.sh "$T2" | jq -c .
{"sub":"42","name":"<<>>?","exp":1767225600}
$ ./jwtpayload.sh 'x.eyJzdWIiOiI0M.y'; echo "exit $?"
b64url_decode: impossible length 13
exit 1
$ ./jwtpayload.sh 'x.eyJ!dWIi.y'; echo "exit $?"
base64: stdin: (null): error decoding base64 input stream
exit 1
```

`T2` is a second made-up token whose payload contains a `-` and needs `=`
padding. I also ran the script on payloads of length % 4 = 0, 2 and 3; all
three came back as complete JSON, whether the script ran under `/bin/sh`,
`zsh`, `/bin/bash` 3.2 or `/bin/dash`.

## Fix 2: jq's `@base64d`

jq 1.7.1 on this Mac (`/usr/bin/jq`) decodes Base64 without the padding by
itself:

```console
$ echo "$TOKEN" | jq -R 'split(".")[1] | @base64d | fromjson' -c
{"sub":"42","role":"admin","exp":1767225600}
```

But it rejects `-` and `_`, so change those first:

```console
$ echo "$T2" | jq -R 'split(".")[1] | @base64d | fromjson' -c
jq: error (at <stdin>:1): string ("eyJzdWIiOi...) is not valid base64 data
$ echo "$T2" | jq -R 'split(".")[1] | gsub("-";"+") | gsub("_";"/") | @base64d | fromjson' -c
{"sub":"42","name":"<<>>?","exp":1767225600}
```

For comparison, the plain `base64 -d` line with `T2` cuts it off again:

```console
$ echo "$T2" | cut -d. -f2 | base64 -d; echo
{"sub":"42","name":"<<>>?","exp":176722560
```

## Fix 3: Python, padding added

```console
$ python3 -c 'import base64, json, sys; p = sys.argv[1].split(".")[1]; print(json.loads(base64.urlsafe_b64decode(p + "=" * (-len(p) % 4))))' "$T2"
{'sub': '42', 'name': '<<>>?', 'exp': 1767225600}
```

`-len(p) % 4` is the number of `=` needed (0, 1 or 2 — or 3 for a broken
length, which Python then rejects). `urlsafe_b64decode` handles `-` and `_`.

All three only *read* the token. None of them checks the signature, so don't
use the result to decide whether to trust the token.

## Checklist

- `base64 -d` on macOS, given Base64 without the `=` padding, drops the last
  incomplete group: 1 or 2 bytes go missing, and the exit status is 0.
- The symptom is JSON cut off at the end: jq "Unfinished JSON term at EOF",
  a missing `}`, a number one digit short.
- JWT parts (and other base64url strings) usually have no padding. Add `==`
  when the length % 4 is 2, `=` when it is 3; 1 means the string is broken.
- Change `-` → `+` and `_` → `/` before handing base64url to a standard
  Base64 decoder; jq's `@base64d` needs it.
- An exit status of 0 from `base64 -d` doesn't mean you got all the bytes.
  Check the length, or let a strict decoder (Python) complain.

Related: [macOS `stat`: file size and `wc -c` padding](/2026/10/macos-stat-illegal-option-c-file-size-mtime-wc-padding/)
and [Python `json.dumps` NaN breaks `JSON.parse`](/2026/10/python-json-dumps-nan-unexpected-token-n-is-not-valid-json/).

*Tested on 2026-11-13 on macOS 26.5.2 (build 25F84, Apple silicon) with
`/usr/bin/base64` (text_cmds-199), `/usr/bin/jq` 1.7.1, `/usr/bin/openssl`
(LibreSSL 3.3.6), Homebrew OpenSSL 3.6.3 (one check), `/usr/bin/python3`
3.8.2, zsh 5.9 and bash 3.2.57;
`PATH=/usr/bin:/bin:/usr/sbin:/sbin`. Not tested: GNU coreutils `base64`,
Linux, older macOS versions.*
