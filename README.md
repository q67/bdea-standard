<p align="center">
  <img alt="spec version" src="https://img.shields.io/badge/spec%20version-2.0-blue">
  <img alt="license" src="https://img.shields.io/badge/license-CC%20BY%204.0-lightgrey">
</p>

# BDEA — the `.bdea` file extension

**`.bdea` is a file extension for BDEA index files** — plain-text,
self-describing catalogs that describe the contents of a long-term backup
archive (optical discs such as Blu-ray or DVD, tape systems such as LTO,
and similar write-once, high-durability media). A `.bdea` file lets an
archive's full logical
structure — volumes, directories, files, duplicates, deletions — be
reconstructed later, without depending on the original computer, database,
or software that produced it.

This repository is the **open specification** for the `.bdea` format. It
defines only the file format itself — not any particular application's
behavior. Anyone may write software that reads or writes `.bdea` files
under this specification, for any purpose, including commercial use, without
asking permission.

> **Current version: 2.0** —
> [specification](versions/2.0/BDEA-2.0_STANDARD.md) ·
> [worked example](versions/2.0/samples/music.bdea)

| Version | Status | Specification | Example |
|---------|--------|---------------|---------|
| **2.0** | **Current** — use for new files | [`BDEA-2.0_STANDARD.md`](versions/2.0/BDEA-2.0_STANDARD.md) | [`music.bdea`](versions/2.0/samples/music.bdea) |
| 1.0 | Superseded — still valid for reading existing 1.0 files | [`BDEA-1.0_STANDARD.md`](versions/1.0/BDEA-1.0_STANDARD.md) | [`music.bdea`](versions/1.0/samples/music.bdea) |

Each version lives at a fixed path under [`versions/`](versions/), so links
to a given version never change when a newer one is released. What changed
between versions is in [`CHANGELOG.md`](CHANGELOG.md). Specifications are in
English and authoritative. Licensed under [CC BY 4.0](LICENSE).

## Why

Optical discs and tape are durable, but the software and databases used to
catalog what's on them are not — file managers change, apps get
discontinued, computers get replaced. BDEA fixes the catalog itself as a
plain-text file that travels with the archive: a copy of the `.bdea` index
is written onto every volume, so any single disc from the set can, on its
own, describe where every file in the whole archive physically lives.

The format is:

- **Self-describing** — field layouts are declared inside the file itself,
  so a parser never has to guess at a schema.
- **Append-only** — new volumes extend the index; nothing already written
  is ever rewritten, which matches how write-once media actually works.
- **Content-addressed** — files are matched by size + hash, not by name or
  path, so duplicates are detectable and a file survives being renamed or
  moved.
- **Verifiable with stock tools** — hashes are MD5 in lowercase hex, the
  same form `md5sum`, `md5`, `certutil` and `Get-FileHash` print, so a
  volume can be checked against its index with nothing but the OS.

## At a glance

```
! ARCHIVE_TYPE BDEA
! VERSION 2.0
! LIBRARY Music
! CONTENT_HASH_ALGORITHM md5
! CONTENT_HASH_ENCODING hex
...

VOLUME_START|1|Music-001|2026-08-13_1232
D|1|0|Artist One
D|2|1|Artist One - Album One
F|1|2|cover.jpg|125142|7fb5d46fb9bff11f638d42c231936c83
F|2|2|...01 Track One.flac|11124049|ef4e7f1fe605a39cd47f2a05f58faee5|tag1,tag2
...
VOLUME_END|1|Music-001|2026-08-13_1232|8|82235912|d83d681ff83f05ceb2348792b9300a01|3
```

See [`BDEA-2.0_STANDARD.md`](versions/2.0/BDEA-2.0_STANDARD.md) for the full grammar, and
[`versions/2.0/samples/music.bdea`](versions/2.0/samples/music.bdea) for a complete worked example
(four volumes, duplicate content, an alias, renames, and recovery from a
failed volume).

## Status

Current format version: **2.0** (see §11 *Versioning* in the spec). This
repository defines the format only. Reference applications
(browser/generator/linter) are a separate, ongoing effort.

## Implementations

No third-party implementations are known yet. Building one? Open a PR to
list it here.

## Contributing

Issues and pull requests proposing clarifications or extensions to the spec
are welcome. Any change to the `! TYPE`/`! RECORD` shapes bumps
`! VERSION` (§11) — a backward-compatible addition bumps `MINOR`, anything
breaking bumps `MAJOR`. Open an issue to discuss before sending a PR that
would require either.

## FAQ

**What does "BDEA" stand for?**
Internally, "Backup Data Extended Archive" — but that's just this project's
own expansion, not something the format depends on. Treat `.bdea` as a file
extension name first; the letters behind it aren't load-bearing.

**Can I use this in a commercial product?**
Yes. The spec is CC BY 4.0 — reuse and build on it for any purpose,
commercial included, as long as you credit this specification as the
source.

**Is there a reference implementation?**
Not yet. The spec is deliberately implementation-agnostic; see
[`BDEA-2.0_STANDARD.md` §10](versions/2.0/BDEA-2.0_STANDARD.md#10-notes-for-implementers)
for notes aimed at anyone building one.

## License

The specification text (everything under `versions/`, and this README) is licensed
under [CC BY 4.0](LICENSE) — share and adapt freely,
including commercially, with attribution.
