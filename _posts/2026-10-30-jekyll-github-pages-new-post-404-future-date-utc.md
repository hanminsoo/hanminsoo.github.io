---
title: "GitHub Pages build succeeds but the new Jekyll post is 404: the file date is \"in the future\" in UTC (`Skipping: ... has a future date`)"
description: >-
  A Jekyll post shows up in a local build, the GitHub Pages build finishes
  without an error, and the post's URL is a 404. Jekyll skips posts dated after
  the build time (`future: false` is the default), and a post named
  `2026-10-30-....md` means midnight on 2026-10-30 in the build machine's time
  zone. GitHub Pages builds in UTC, so from 00:00 to 09:00 in Korea (UTC+9)
  a post named with the local date is still in the future there. Reproduced
  with Jekyll 3.10.0 and `TZ=UTC`: exit 0, no warning, the post is only
  mentioned under `--verbose`. Fixes: name the file with `date -u +%F`, or
  set `timezone:` (which can move existing URLs), or `future: true` (which
  publishes scheduled posts too). Also: an unquoted front matter `date:`
  without an offset was read as UTC here.
tags: [jekyll, github-pages, timezone, utc, static-site]
---

I named a new post with today's date, pushed it, and the GitHub Pages
build finished without an error. The post's URL was a 404. A build on
my Mac, which runs in Korean time, had the post. Nothing in the build
said that a post was left out.

The cause: Jekyll doesn't publish posts dated after the time of the build,
and GitHub Pages builds in UTC. It was before 9 a.m. in Korea, so in UTC
the file's date was still tomorrow.

Below I reproduce it with a local Jekyll by changing only `TZ`, show the one
place where Jekyll mentions the skipped post, and test three fixes. One of
them can move the URLs of posts that are already published.

## Why the date is "in the future"

Two rules work together here:

1. **Jekyll skips future posts by default.** The `future` setting is `false`
   unless you change it. A post whose date is later than the build time
   (`site.time`) is left out of the site.
2. **A file name like `2026-10-30-new-post.md` has no time and no time
   zone.** Jekyll takes it as midnight at the start of that day, in the time zone of
   the machine running the build.

Korea is UTC+9. At 08:00 in Seoul on 2026-10-30 it is 23:00 on 2026-10-29
in UTC. A build on a UTC machine reads `2026-10-30` as
2026-10-30 00:00 UTC, which is an hour ahead of "now". It skips the post. A
build on my Mac reads the same name as 2026-10-30 00:00 in Seoul, which was
eight hours earlier, so it publishes the post.

How do I know Pages builds in UTC? The feed of this blog, built by GitHub
Pages, has the build time in it, and the offset is `+00:00` (the time
itself is from the last build before I checked):

```console
$ curl -s https://hanminsoo.github.io/feed.xml | grep -o '<updated>[^<]*</updated>' | head -1
<updated>2026-10-29T01:06:21+00:00</updated>
```

(jekyll-feed writes the build time there. This blog has no `timezone:` in
`_config.yml`, so the offset is the build machine's own.)

## Reproduce it

You need a working `jekyll` command (`bundle exec jekyll` in most setups).
The site has two posts' worth of files and nothing else.

`_clock.yml` sets `time:`, a Jekyll config key that overrides the build
time (`site.time`). I use it so that the result doesn't depend on when you
run this: the build "happens" at 08:00 in Seoul on 2026-10-30. Pages uses
the real clock, so you hit the same thing between 00:00 and 09:00 Korean
time without this file.

```console
$ mkdir demo && cd demo
$ mkdir _posts
$ printf 'title: Demo\npermalink: /:year/:month/:day/:title/\n' > _config.yml
$ printf -- '---\ntitle: Home\n---\nHome\n' > index.md
$ printf -- '---\ntitle: New post\n---\nDate: {{ page.date }}\n' > _posts/2026-10-30-new-post.md
$ printf 'time: 2026-10-30 08:00:00 +0900\n' > _clock.yml
```

Build it in Korean time, as my Mac does:

```console
$ TZ=Asia/Seoul jekyll build -q --config _config.yml,_clock.yml; echo "exit $?"
exit 0
$ find _site -name '*.html' | sort
_site/2026/10/30/new-post/index.html
_site/index.html
$ cat _site/2026/10/30/new-post/index.html
<p>Date: 2026-10-30 00:00:00 +0900</p>
```

The same files, built in UTC, as Pages does:

```console
$ TZ=UTC jekyll build -q --config _config.yml,_clock.yml; echo "exit $?"
exit 0
$ find _site -name '*.html' | sort
_site/index.html
```

Exit 0 and no warning. The post is gone, so its URL is a 404. The page that
the first build wrote is gone too, because Jekyll clears out files it didn't
write this time.

The only mention of the skipped post is a debug line, shown with `--verbose`:

```console
$ TZ=UTC jekyll build --verbose --config _config.yml,_clock.yml | grep -i skip
          Skipping: _posts/2026-10-30-new-post.md has a future date
```

On the default GitHub Pages build I didn't get `--verbose`. In my case the build showed as
finished without an error, and only the new post was missing.

## Fix 1: name the file with the UTC date

The fix with the fewest side effects is to date the file by UTC, the clock that
Pages uses. When you create the file:

```bash
touch "_posts/$(date -u +%F)-new-post.md"
```

With the name changed to the UTC date at that moment (2026-10-29), the UTC
build publishes it:

```console
$ mv _posts/2026-10-30-new-post.md _posts/2026-10-29-new-post.md
$ TZ=UTC jekyll build -q --config _config.yml,_clock.yml
$ find _site -name '*.html' | sort
_site/2026/10/29/new-post/index.html
_site/index.html
$ cat _site/2026/10/29/new-post/index.html
<p>Date: 2026-10-29 00:00:00 +0000</p>
```

The post's date and URL now say the 29th, which was "yesterday" in Korea.
I was fine with that. If you aren't, see fix 2.

## Fix 2: `timezone:` in `_config.yml`

`timezone:` makes Jekyll read dates in the zone you name, whatever zone
the build machine uses. With the file back at the Korean date:

```console
$ mv _posts/2026-10-29-new-post.md _posts/2026-10-30-new-post.md
$ printf 'timezone: Asia/Seoul\n' >> _config.yml
$ TZ=UTC jekyll build -q --config _config.yml,_clock.yml
$ find _site -name '*.html' | sort
_site/2026/10/30/new-post/index.html
_site/index.html
$ cat _site/2026/10/30/new-post/index.html
<p>Date: 2026-10-30 00:00:00 +0900</p>
```

I tested this with a local `TZ=UTC` build, not on GitHub Pages itself.

**It can change URLs that are already published.** A post whose front
matter has a time with an offset is a fixed moment. A new zone can put
that moment on another day, and a permalink that contains the date then
changes. Starting from the folder that contains `demo`:

```console
$ cd .. && mkdir demo3 && cd demo3
$ mkdir _posts
$ printf 'title: Demo\npermalink: /:year/:month/:day/:title/\n' > _config.yml
$ printf -- '---\ntitle: Late\ndate: 2026-10-28 23:30:00 +0000\n---\n{{ page.date }}\n' > _posts/2026-10-28-late.md
$ TZ=UTC jekyll build -q
$ find _site -name '*.html'
_site/2026/10/28/late/index.html
$ printf 'timezone: Asia/Seoul\n' >> _config.yml
$ TZ=UTC jekyll build -q
$ find _site -name '*.html'
_site/2026/10/29/late/index.html
$ cat _site/2026/10/29/late/index.html
<p>2026-10-29 08:30:00 +0900</p>
```

`/2026/10/28/late/` is now a 404 and the post is at `/2026/10/29/late/`.
Posts dated only by their file name stay on the same day, as `new-post`
did above. Before you add `timezone:` to an existing site, look for posts
with a `date:` in the front matter.

## Fix 3: `future: true` (or `--future`)

This turns the check off. It also publishes posts that you dated ahead on
purpose:

```console
$ cd .. && mkdir demo2 && cd demo2
$ mkdir _posts
$ printf 'title: Demo\npermalink: /:year/:month/:day/:title/\n' > _config.yml
$ printf 'time: 2026-10-30 08:00:00 +0900\n' > _clock.yml
$ printf -- '---\ntitle: A\n---\n{{ page.date }}\n' > _posts/2026-10-30-today-kst.md
$ printf -- '---\ntitle: B\n---\n{{ page.date }}\n' > _posts/2026-12-25-scheduled.md
$ TZ=UTC jekyll build -q --future --config _config.yml,_clock.yml
$ find _site -name '*.html' | sort
_site/2026/10/30/today-kst/index.html
_site/2026/12/25/scheduled/index.html
```

The December post is out in October. If you never date posts ahead, this
is a simple fix. If you schedule posts that way, don't use it. On Pages you
would put `future: true` in `_config.yml`. I didn't test it there.

## Front matter `date:`: quote it or give an offset

The same thing can happen with a `date:` in the front matter. Here I set
`timezone: Asia/Seoul` and wrote 09:00 three ways:

```console
$ cd .. && mkdir demo4 && cd demo4
$ mkdir _posts
$ printf 'title: Demo\ntimezone: Asia/Seoul\n' > _config.yml
$ printf -- '---\ntitle: P\ndate: 2026-10-30 09:00:00\n---\n{{ page.date }}\n' > _posts/2026-10-30-plain.md
$ printf -- '---\ntitle: Q\ndate: "2026-10-30 09:00:00"\n---\n{{ page.date }}\n' > _posts/2026-10-30-quoted.md
$ printf -- '---\ntitle: O\ndate: 2026-10-30 09:00:00 +0900\n---\n{{ page.date }}\n' > _posts/2026-10-30-offset.md
$ jekyll build -q --future
$ cat _site/2026/10/30/plain.html _site/2026/10/30/quoted.html _site/2026/10/30/offset.html
<p>2026-10-30 18:00:00 +0900</p>
<p>2026-10-30 09:00:00 +0900</p>
<p>2026-10-30 09:00:00 +0900</p>
```

Unquoted and without an offset, `09:00:00` was read as 09:00 **UTC**
(18:00 in Seoul), even with `timezone:` set. A post published "at 9 a.m."
that way stays in the future for nine more hours. Quoted,
it was read in the site's zone. With `+0900`, it means one moment wherever
it's built. I saw this with Ruby 2.6. GitHub Pages runs Jekyll on a
newer Ruby (3.3.4 per pages.github.com/versions.json) and I didn't check it
there, so the safe habit is to always write the offset.

(`--future` is there so that the 18:00 post isn't skipped. I built this
one with the real clock.)

## Check before you push

A small check you can run before `git push`. It lists posts whose file
name date is later than today in UTC:

```sh
#!/bin/sh
# check-future.sh -- list posts whose file-name date is after today in UTC
today=$(date -u +%F)
status=0
for f in _posts/*; do
  d=$(basename "$f" | cut -c1-10)
  if [ "$d" \> "$today" ]; then
    echo "future in UTC (today is $today): $f"
    status=1
  fi
done
exit $status
```

It compares the names as text, which works because `YYYY-MM-DD` sorts the
same way as dates. It doesn't read front matter dates. With `check-future.sh`
in the folder above `demo5`:

```console
$ cd .. && mkdir demo5 && cd demo5
$ mkdir _posts
$ touch "_posts/$(date -u +%F)-today.md" "_posts/$(date -u -v+1d +%F)-tomorrow.md"
$ sh ../check-future.sh; echo "exit $?"
future in UTC (today is 2026-10-30): _posts/2026-10-31-tomorrow.md
exit 1
$ rm _posts/*-tomorrow.md
$ sh ../check-future.sh; echo "exit $?"
exit 0
```

(The dates are from the day I ran it. `date -v+1d` is the macOS way to say
"tomorrow". On Linux, use `date -u -d tomorrow +%F`, which I didn't test.)

## Checklist

- New post 404 on GitHub Pages, build fine, local build has it: compare the
  file name date with `date -u +%F`.
- Jekyll skips posts dated after the build time by default (`future: false`).
  No error, exit 0. Only `--verbose` prints `Skipping: ... has a future date`.
- A file name date means midnight in the build machine's zone. Pages
  builds in UTC (this blog's feed says `+00:00`). East of UTC, a post named
  with your local date can be "tomorrow" there for several hours (UTC+9:
  00:00–09:00).
- To reproduce locally: `TZ=UTC jekyll build --verbose`.
- Fix 1: name posts with `date -u +%F`.
- Fix 2: `timezone:` in `_config.yml`. It can move posts with a front matter
  time to another day, and their URLs with them.
- Fix 3: `future: true`. It publishes scheduled posts too.
- In front matter, write `date: 2026-10-30 09:00:00 +0900`. Without
  quotes or an offset, it was read as UTC here.

*Tested on 2026-10-30 on macOS 26.5.2 (build 25F84, Apple silicon) with
Jekyll 3.10.0 (the version GitHub Pages lists and this blog's feed shows) on the
system Ruby 2.6.10, in a scratch folder, with `TZ=Asia/Seoul` and `TZ=UTC` to
stand in for my Mac and for Pages. The gems were installed into a scratch folder without
jekyll-watch and em-websocket, so `jekyll serve` was not tested. Not tested
on GitHub Pages itself: `timezone:`, `future: true`, and the unquoted front
matter date on Ruby 3.3. Linux not tested.*
