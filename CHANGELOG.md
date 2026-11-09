# Changelog

All notable changes to the BDEA format specification are recorded here.
This tracks the *format* (`! VERSION`), not the applications that
implement it.

## [2.0] — 2026-11-08

Breaking change to how content hashes are written. Specification:
`versions/2.0/BDEA-2.0_STANDARD.md`; worked example:
`versions/2.0/samples/music.bdea`.

- `! CONTENT_HASH_ENCODING` changes from `base32` to `hex`: lowercase
  hexadecimal, no separators or padding — 32 characters for MD5 instead of
  26. This is the form stock OS tools print (`md5sum`, `md5`, `certutil`,
  `Get-FileHash`), so a recorded hash can be checked without any
  BDEA-specific software; base32 MD5 can't be produced by any of them
  directly (§5 "Content hash encoding").
- `hash`/`volume_hash` values MUST be lowercase; readers comparing against
  external tool output should compare case-insensitively.
- `volume_hash` keeps its 1.0 definition (hash of the concatenated `hash`
  text of a volume's `F` records), now spelled out as taking the lowercase
  hex text's ASCII bytes — reproducible with
  `printf '%s' HASH1 HASH2 ... | md5sum` (§7 `VOLUME_END`).
- `! TYPE`/`! RECORD` layouts are otherwise unchanged from 1.0.
- New §4 "Characters": a line break is LF or CR LF; "space" means the
  space character only; apart from line breaks the file holds only
  printable characters and spaces (no tabs); no record field — not only
  names — may contain `|`, which declarations may still use. These hold
  whatever `! NAME_FORBIDDEN_CHARACTERS` declares (§5); §10 notes that
  source names which can't be recorded as-is must not be written
  unchanged.
- 1.0 remains documented in `versions/1.0/BDEA-1.0_STANDARD.md` with
  its example in `versions/1.0/samples/music.bdea`. A 1.0 file
  converts to 2.0 mechanically — re-encode each `hash` from base32 to hex,
  recompute each `volume_hash`, update `! VERSION` and
  `! CONTENT_HASH_ENCODING` (§11).

## [1.0] — 2026-08-24

Initial public release.

- `! VERSION` is `MAJOR.MINOR`; see
  `versions/1.0/BDEA-1.0_STANDARD.md` §11 for what qualifies as a
  minor (backward-compatible) vs. a major (breaking) change.
- `! TYPE`/`! RECORD` schema as specified in
  `versions/1.0/BDEA-1.0_STANDARD.md` §6–§7: `VOLUME_START`/
  `VOLUME_END`, `AF`, `XF`, `XD`, `RD`, `RF`, `D`, `F`, `S`.
- Index filenames only require the `.bdea` extension — the rest of the
  name is unconstrained; the `! LIBRARY` header, not the filename,
  identifies which library a file describes.
