# BDEA-Standard/ — BDEA format definition

Source of truth for the `.bdea` index format.

- [`BDEA_STANDARD.md`](BDEA_STANDARD.md) — full specification. Read this
  first, before touching `.bdea` files or code that parses/writes them.
  [`BDEA_STANDARD.ua.md`](BDEA_STANDARD.ua.md) is a Ukrainian translation of
  the same content (English is authoritative on any discrepancy).
- `samples/music.bdea` — canonical worked example: a self-describing header
  plus two volumes, including a duplicate-content case and an active/deleted
  correction. Only the `.bdea` extension is required by the format — this
  sample happens to be named `music.bdea` after its `! LIBRARY` (`Music`),
  but the filename itself carries no meaning under the standard.

See the root [`CLAUDE.md`](../CLAUDE.md) for project-wide conventions
(communicate with the user in Ukrainian).
