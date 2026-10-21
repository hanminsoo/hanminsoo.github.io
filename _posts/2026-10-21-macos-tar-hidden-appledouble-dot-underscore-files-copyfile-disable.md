---
title: "macOS tar adds hidden `._` files that `tar tf` doesn't show: `COPYFILE_DISABLE=1`, `--no-mac-metadata`, `--no-xattrs` (and `__MACOSX` in zips)"
description: >-
  In my test, a file with one extended attribute I added by hand made macOS
  tar (bsdtar 3.5.3) write an extra `._name` AppleDouble entry plus pax xattr
  headers into the archive. macOS's own `tar tzf` doesn't list the `._` entry,
  but Python's tarfile does, and extracting with Python's tarfile created it
  as a real file (other extractors not tested). `COPYFILE_DISABLE=1` or
  `--no-mac-metadata` drops the `._` entry but keeps the xattr headers.
  `--no-xattrs` drops the headers but keeps `._`. You need both. Zips made with
  `ditto -c -k --sequesterRsrc` get a `__MACOSX/` folder; `zip -r` and plain
  `ditto -c -k` didn't. Reproduced on macOS 26.
tags: [macos, tar, zip, xattr, shell]
---

You make a tarball on a Mac, and it contains an entry you never made,
`._index.html` next to `index.html`. When I extracted it with Python's
`tarfile`, that entry came out as a real file. I didn't test other
extractors. On the Mac, the archive looks clean:

```console
$ mkdir demo && cd demo
$ mkdir site && echo hi > site/index.html && echo b > site/app.js
$ xattr -w com.example.note hello site/index.html
$ tar czf site.tgz site
$ tar tzf site.tgz
site/
site/index.html
site/app.js
```

The extra entry is there; macOS tar just doesn't show it. Python's
`tarfile`, which knows nothing about Macs, lists what is actually in the
file:

```console
$ python3 -m tarfile -l site.tgz
site/ 
site/._index.html 
site/index.html 
site/app.js 
```

Below: where the `._` comes from, which flag removes what (it takes two),
and the same thing in zip files.

## Where it comes from

`site/index.html` has an extended attribute. Here I added one by hand.
`ls -l` marks such files with `@`, and `xattr -l` shows them:

```console
$ xattr -l site/index.html
com.example.note: hello
$ tar --version
bsdtar 3.5.3 - libarchive 3.7.4 zlib/1.2.12 liblzma/5.4.3 bz2lib/1.0.8 
```

For a file like that, macOS tar stores the Mac metadata **twice**: as an
AppleDouble `._index.html` entry, and as pax headers on `index.html`
itself. Python shows the headers too:

```console
$ python3 -c 'import sys,tarfile; t=tarfile.open(sys.argv[1]); [print(m.name, sorted(m.pax_headers)) for m in t]' site.tgz
site []
site/._index.html []
site/index.html ['LIBARCHIVE.xattr.com.example.note', 'SCHILY.xattr.com.example.note', 'mtime']
site/app.js []
```

`app.js` has no attribute and gets neither. In this test, only the file
with an attribute got the extra entry.

## Which flag removes what

I made the same archive with each option and listed it with the same
Python line:

```console
$ COPYFILE_DISABLE=1 tar czf a.tgz site
$ tar --no-mac-metadata -czf b.tgz site
$ tar --no-xattrs -czf c.tgz site
$ COPYFILE_DISABLE=1 tar --no-xattrs -czf d.tgz site
$ for f in a b c d; do echo "== $f"; python3 -c 'import sys,tarfile; t=tarfile.open(sys.argv[1]); [print(m.name, sorted(m.pax_headers)) for m in t]' $f.tgz; done
== a
site []
site/index.html ['LIBARCHIVE.xattr.com.example.note', 'SCHILY.xattr.com.example.note', 'mtime']
site/app.js []
== b
site []
site/index.html ['LIBARCHIVE.xattr.com.example.note', 'SCHILY.xattr.com.example.note', 'mtime']
site/app.js []
== c
site []
site/._index.html []
site/index.html []
site/app.js []
== d
site []
site/index.html []
site/app.js []
```

- `COPYFILE_DISABLE=1` and `--no-mac-metadata` remove the `._` entry, but
  the attribute is still in the pax headers.
- `--no-xattrs` removes the pax headers, but the `._` entry is still there.
- Only both together (`d`) give a plain archive.

Whether the leftover pax headers matter depends on the tool that reads
the archive. I didn't test GNU tar or a Linux extract, so I can't say
what they do with `LIBARCHIVE.xattr.*`. The `._` file, on the other hand,
is an ordinary tar entry. Python's extractor doesn't know about
AppleDouble and writes it out as a file. I only tested Python:

```console
$ mkdir ex && python3 -m tarfile -e site.tgz ex && find ex | sort
ex
ex/site
ex/site/._index.html
ex/site/app.js
ex/site/index.html
```

## Zip: `__MACOSX/` from ditto

A zip made with `ditto -c -k --sequesterRsrc` gets the same kind of
metadata, as a `__MACOSX/` folder:

```console
$ ditto -c -k --sequesterRsrc --keepParent site site-d.zip; unzip -l site-d.zip | grep MACOSX
        0  10-21-2026 12:19   __MACOSX/
        0  10-21-2026 12:19   __MACOSX/site/
      153  10-21-2026 12:19   __MACOSX/site/._index.html
$ ditto -c -k --keepParent site site-p.zip; unzip -l site-p.zip | grep -c MACOSX
0
$ zip -qr site.zip site; unzip -l site.zip | grep -c MACOSX
0
```

Of these three, only `--sequesterRsrc` added it in my test. (Dates and
times in `unzip -l` are from my run.) I didn't check what Finder's
"Compress" uses.

Extracting it with `unzip` gives you a `__MACOSX` folder and doesn't
restore the attribute. `ditto -x -k` restores it instead:

```console
$ mkdir uz && (cd uz && unzip -q ../site-d.zip) && find uz -name '._*'; xattr -l uz/site/index.html
uz/__MACOSX/site/._index.html
$ mkdir dx && ditto -x -k site-d.zip dx && find dx -name '._*'; xattr -l dx/site/index.html
com.example.note: hello
```

To strip it from an existing zip:

```console
$ zip -d site-d.zip '__MACOSX/*'
deleting: __MACOSX/
deleting: __MACOSX/site/
deleting: __MACOSX/site/._index.html
```

## Or drop the attributes first

If you don't need the attributes, remove them, and the default `tar`
has nothing extra to write:

```console
$ xattr -c site/index.html && tar czf e.tgz site && python3 -m tarfile -l e.tgz
site/ 
site/index.html 
site/app.js 
```

## Checklist

- In this test, one extended attribute added by hand (`xattr -w`) was
  enough for macOS tar to add a hidden `._name` entry plus pax xattr headers.
  I didn't test other attributes.
- macOS `tar tf` doesn't list the `._` entry. Check with
  `python3 -m tarfile -l FILE.tgz`.
- `COPYFILE_DISABLE=1` or `--no-mac-metadata`: no `._`, headers stay.
- `--no-xattrs`: no headers, `._` stays.
- For a plain archive: `COPYFILE_DISABLE=1 tar --no-xattrs -czf ...`.
- Zips: `ditto -c -k --sequesterRsrc` adds `__MACOSX/`. `zip -r` and plain
  `ditto -c -k` didn't here. Strip with `zip -d FILE.zip '__MACOSX/*'`.
- Or remove the attributes first (`xattr -c FILE`), if you don't need them.

*Tested on macOS 26.5.2 (Apple silicon) with `/usr/bin/tar` (`bsdtar 3.5.3 -
libarchive 3.7.4`), `/usr/bin/zip` (`Zip 3.0 (July 5th 2008), by Info-ZIP, with
modifications by Apple Inc.`), and `/usr/bin/python3` (Python 3.8.2), in zsh 5.9
(arm64-apple-darwin25.0). GNU tar, Linux, and Finder's Compress were not tested.*
