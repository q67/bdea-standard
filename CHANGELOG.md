# Changelog

All notable changes to the BDEA format specification are recorded here.
This tracks the *format* (`! VERSION`), not the applications that
implement it.

## [1.0] — 2026-08-24

Initial public release.

- `! VERSION` is `MAJOR.MINOR`; see `BDEA_STANDARD.md` §11 for what
  qualifies as a minor (backward-compatible) vs. a major (breaking) change.
- `! TYPE`/`! RECORD` schema as specified in `BDEA_STANDARD.md` §6–§7:
  `VOLUME_START`/`VOLUME_END`, `D`, `F`, `S`, `AF`, `XF`, `XD`, `RD`, `RF`.
- Index filenames only require the `.bdea` extension — the rest of the
  name is unconstrained; the `! LIBRARY` header, not the filename,
  identifies which library a file describes.
