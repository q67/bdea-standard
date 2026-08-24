# Changelog

All notable changes to the BDEA format specification are recorded here.
This tracks the *format* (`! VERSION`), not the applications that
implement it.

## [1] — 2026-08-24

Initial public release.

- `! TYPE`/`! RECORD` schema as specified in `BDEA_STANDARD.md` §6–§7:
  `VOLUME_START`/`VOLUME_END`, `D`, `F`, `S`, `A`, `X`.
- Index filenames only require the `.bdea` extension — the rest of the
  name is unconstrained; the `! LIBRARY` header, not the filename,
  identifies which library a file describes.
