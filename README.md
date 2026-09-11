<p align="center">
  <img alt="spec version" src="https://img.shields.io/badge/spec%20version-1.0-blue">
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

- **Specification:** [`BDEA-1.0_STANDARD.md`](BDEA-1.0_STANDARD.md) (English,
  authoritative)
- **Worked example:** [`samples/music.bdea`](samples/music.bdea)
- **License:** [CC BY 4.0](LICENSE)

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

## At a glance

```
! ARCHIVE_TYPE BDEA
! VERSION 1.0
! LIBRARY Music
...

VOLUME_START|1|Music-001|2026-08-13_1232

D|1|0|Music
D|2|1|Rafael Krux
F|1|3|cover.jpg|125142|P625I35ZX7YR6Y4NILBDDE3MQP
F|2|3|...02 Classical Violin Concerto.flac|8541907|VUMGOS3L25V5JEP4LU6OI5PXQG|baroque,epic

VOLUME_END|1|Music-001|2026-08-13_1232|9|0|0|9|70477923|6VJWG7TG73DXOYCYXY6R6UXYUU|4
```

See [`BDEA-1.0_STANDARD.md`](BDEA-1.0_STANDARD.md) for the full grammar, and
[`samples/music.bdea`](samples/music.bdea) for a complete worked example
(two volumes, a duplicate-content case, and an active/deleted correction).

## Status

Current format version: **1.0** (see §11 *Versioning* in the spec). This
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
[`BDEA-Standard/BDEA-1.0_STANDARD.md` §10](BDEA-1.0_STANDARD.md#10-notes-for-implementers)
for notes aimed at anyone building one.

## License

The specification text (`BDEA-1.0_STANDARD.md` and this README) is licensed
under [CC BY 4.0](LICENSE) — share and adapt freely,
including commercially, with attribution.
