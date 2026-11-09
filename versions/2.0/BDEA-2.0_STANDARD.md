# BDEA Index Format Standard

**BDEA — Backup Data Extended Archive.** A long-term, offline archive format for
write-once/high-durability media (optical discs such as Blu-ray or DVD, tape
systems such as LTO, and similar). This document specifies the `.bdea` index
file format. It does not
specify any particular application's behavior (see [§10](#10-notes-for-implementers)).

This is the authoritative specification for `! VERSION 2.0`. See
[`samples/music.bdea`](samples/music.bdea) for a canonical worked example.
Files declaring `! VERSION 1.0` are described by
[`versions/1.0/BDEA-1.0_STANDARD.md`](../1.0/BDEA-1.0_STANDARD.md) instead (§11).

## 1. Purpose and scope

BDEA's goal is to let a **library** of archived files be reconstructed later —
its full logical structure, not just raw file blobs — without depending on the
original computer, database, or software. The standard covers **only** the
`.bdea` index file format: how a library's structure, volumes, directories,
files, and their history are described in text. It does not cover physical
medium layout, media-writing software, or backup policy.

## 2. Core concepts

| Term | Meaning |
|---|---|
| **Archive** | The overall BDEA setup: one or more independent libraries. |
| **Library** | A logically isolated collection of data (e.g. `Music`, `Documents`, `Photos`). Has its own directory namespace, its own `.bdea` index, and can be restored independently of any other library. |
| **Volume** | One physical medium (e.g. one disc or tape) belonging to a library. A library grows by adding volumes over time (`Music-001`, `Music-002`, ...). |

**Libraries are fully isolated from each other.** A library never references
another library, duplicates are never detected across libraries, and each
library has its own rules and its own recovery path. BDEA is a set of
autonomous per-library structures, not one monolithic archive.

## 3. File identity model

Two distinct notions of "identity" are used, deliberately kept separate:

- **Physical/positional identity** — `(volume_id, file_id)`. Identifies one
  archived-file *record* inside the library: which volume it was written on
  and its local id within that volume. `file_id` is unique only within its
  `volume_id`.
- **Content identity** — `size` + `hash`. Identifies *what the file actually
  is*, independent of its name or location. Two records with the same
  `size`+`hash` are the same content, wherever they appear.

Consequences of using content identity instead of the file name:

- Duplicate content anywhere in the library can be detected (by comparing
  `size`+`hash` across all `F` records already in the index).
- Content integrity can be verified against the recorded hash.
- A file can move between directories, or even be re-added under a different
  name, without losing its identity.

The name is never authoritative — it's metadata, not identity.

## 4. `.bdea` index file structure overview

**File naming.** A library's index file must have the `.bdea` extension;
the rest of the filename is arbitrary — nothing in this standard requires
it to match `! LIBRARY`, its casing, or otherwise encode which library the
file describes. A reader determines that from the `! LIBRARY` header
declaration (§5) inside the file, not from the filename. One file per
library — a file could equally be named `music.bdea`, `music-2026-08-23.bdea`,
or anything else ending in `.bdea`; an application must never infer the
library from the name.

A `.bdea` file is UTF-8 text with three kinds of lines:

- **Declarations** — start with `!`, whitespace-separated tokens:
  `! KEYWORD value...`. These are machine-readable.
- **Comments** — start with `#`, free text, ignored by parsers.
- **Records** — data lines shaped `TAG|field1|field2|...`, pipe-delimited.
  `TAG` is whatever tag a `! RECORD` declaration (§7) defined — one or more
  characters (`VOLUME_START`, `VOLUME_END`, `AF`, `XF`, `XD`, `RD`, `RF`,
  `D`, `F`, `S` in the current schema).

Blank lines are allowed as visual separators (e.g. between volume sections).

**Field content.** The format has no escaping or quoting mechanism: a
record is split on every `|`, and the file is split into lines on every
line break. So that this splitting is always unambiguous, the following
characters MUST NOT appear in any field of a record (every field, of
every tag — names, `extra`, `volume_label`, everything):

| Characters | Why |
|---|---|
| `\|` (U+007C) | The record field separator. |
| U+0000–U+001F (C0 controls, incl. tab, LF, CR) | LF/CR end the line; the rest are invisible and unsafe in plain text. |
| U+007F (DEL) | Control character. |
| U+0080–U+009F (C1 controls, incl. NEL U+0085) | Control characters; NEL is treated as a line break by some tools. |
| U+2028, U+2029 (line/paragraph separator) | Treated as line breaks by some tools. |

This applies regardless of `! NAME_FORBIDDEN_CHARACTERS` (§5), which lists
characters forbidden in names *in addition* to these.

Declaration lines (§5, §6, §7) are split into tokens on whitespace, not
on `|`, so `|` MAY appear in a declaration value — the canonical
`! NAME_FORBIDDEN_CHARACTERS` value lists it, for instance. Every other
character in the table above MUST NOT appear in a declaration value, for
the same line-splitting reason. Comment lines are free text up to the end
of the line and aren't restricted beyond not containing a line break. A field that would need one
of these characters (e.g. a source file whose name contains a line break,
possible on Linux and macOS) cannot be recorded as-is; what to do about it
is up to the producing application (§10).

**The format is self-describing.** The field order for each record tag is not
hard-coded by this spec — it's declared in the file itself via `! RECORD`
lines (§7), which must appear before any data line using that tag. A
conforming parser reads the `! TYPE`/`! RECORD` declarations first, then uses
that mapping to parse subsequent data lines.

**The index is append-only and cumulative.** The same
`.bdea` file is *appended to* every time a new volume is produced, and the
updated file is written onto that new volume too. This means:

- Every volume's copy of the index contains the *entire library history up to
  that point* — not just that volume's own contents.
- The newest volume you have always holds the most complete/up-to-date index
  for the whole library (see §8 for the consequence of this on write-once
  media).
- A library can, in principle, be browsed in full from a single volume's copy
  of the index, even without the other physical volumes present — that copy
  will say which volume physically holds any given file's bytes.

**Record order is significant.** The logical state of a BDEA library is
resolved by processing records in their order of appearance in the `.bdea`
file — this full, ordered sequence of records, spanning every volume
section, is referred to throughout this document as the log. Within a
volume section, each record updates the reader's working state
immediately, in the order it appears — a correction affects the
state from its position in the log onward, and a later correction to the
same directory or file simply supersedes an earlier one; records unrelated
to a given correction are never affected by it. This is also why a
correction and the new data it concerns can be interleaved freely within a
volume section (§8) instead of being grouped apart from each other — the
only hard requirement is that a record never references something that
doesn't already exist earlier in the log. A reader never needs to
look ahead or rescan to find "the latest" correction — whichever one it
applied most recently to a given target is, by construction, the current
one. Across volume sections, a section's updates become part of the
library's state only once that section's `VOLUME_END` is confirmed valid
(§7 `VOLUME_START`/`VOLUME_END`); a section that never reaches a valid
`VOLUME_END` contributes nothing. See §10 for the full reading algorithm.

## 5. Header declarations

These appear once, near the top of the file, before any volume section.

| Declaration | Example | Meaning |
|---|---|---|
| `! TEXT_ENCODING` | `utf-8` | Encoding of the `.bdea` file itself. |
| `! TEXT_NORMALIZATION` | `NFC` | Unicode normalization form used for the file's text. |
| `! ARCHIVE_TYPE` | `BDEA` | Format identifier. |
| `! VERSION` | `2.0` | Format version (see §11). |
| `! LIBRARY` | `Music` | Name of this library. |
| `! NAME_ENCODING` | `utf-8` | Encoding used for `dir_name`/`file_name` fields. |
| `! NAME_NORMALIZATION` | `NFC` | Unicode normalization applied to names. |
| `! NAME_FORBIDDEN_CHARACTERS` | `/ \ : * ? " < > \|` | Characters names must not contain (superset of Windows/exFAT-reserved characters — a safe choice for media meant to be readable from any filesystem). Adds to, never replaces, the characters forbidden in every field (§4 "Field content"): `\|` and control characters are forbidden in names even if a file's declared list omits them. |
| `! CONTENT_HASH_ALGORITHM` | `md5` | Hash algorithm used to identify file content. |
| `! CONTENT_HASH_ENCODING` | `hex` | Encoding used to render the hash as text: lowercase hexadecimal (see below). |
| `! CONTENT_HASH_SCOPE` | `file-content` | What is hashed (the raw file bytes, as opposed to e.g. name+content). |

This is the canonical, current form of the header, and what new `.bdea`
files should emit.

**Content hash encoding.** `hex` means each byte of the digest is written
as two hexadecimal digits, most significant first, using only the
characters `0-9a-f` — lowercase, no separators, no prefix, no padding. An
MD5 digest therefore always renders as exactly 32 characters, e.g.
`7fb5d46fb9bff11f638d42c231936c83`. This is deliberately the same form the
stock tools of every common operating system already print, so a file's
recorded `hash` can be checked against the archived bytes years later with
nothing but what ships with the OS:

| OS | Command | Output |
|---|---|---|
| Linux | `md5sum FILE` | lowercase hex |
| macOS | `md5 -q FILE` | lowercase hex |
| Windows | `certutil -hashfile FILE MD5` | hex |
| Windows | `Get-FileHash -Algorithm MD5 FILE` (PowerShell) | UPPERCASE hex |

A producing application MUST write `hash`/`volume_hash` values in
lowercase. A reader comparing a recorded value against an external tool's
output SHOULD compare case-insensitively, since some tools (e.g.
PowerShell's `Get-FileHash`) print uppercase; inside the index itself the
lowercase form is the only valid one, because `volume_hash` is computed
over the recorded text (§7 `VOLUME_END`) and would come out different for
the same digest written in another case.

## 6. Type system (`! TYPE`)

Declares the primitive type of every field used by records, so the file is
self-describing and independently parseable. Every `string` value is
additionally subject to the characters forbidden in every field (§4
"Field content").

| Type name | Kind | Notes |
|---|---|---|
| `dir_id` | `uint32` | Global across the library — assigned once per directory and never reused, regardless of which volume introduced it. `0` always denotes the library's root, which needs no `D` record of its own (it has no name or parent to declare); every actual directory is numbered from `1` upward, in the order it's introduced (§7 `D`, `F`, `S`). |
| `last_dir_id` | `uint32` | Same kind as `dir_id`; highest `dir_id` assigned in the library as of a given volume (§7 `VOLUME_END`). |
| `new_dir_id` | `uint32` | Same kind as `dir_id`; the corrected directory an `RF` correction files a record under (§7 `RF`). |
| `parent_dir_id` | `uint32` | The containing (parent) directory's `dir_id`; `0` = root of the library's namespace. |
| `new_parent_dir_id` | `uint32` | Same kind as `parent_dir_id`; the corrected parent an `RD` correction gives a directory (§7 `RD`). |
| `file_id` | `uint64` | Local to a volume. |
| `target_file_id` | `uint64` | Same kind as `file_id`; the `F` record an `S` alias reaches for (§7 `S`). |
| `size` | `uint64` | Bytes. |
| `dir_name` | `string max=120` | Directory name. |
| `file_name` | `string max=120` | File name; also used by `S` aliases, which occupy a file-tree position of their own (§7 `S`). |
| `new_dir_name` | `string max=120` | Same kind as `dir_name`; the corrected name an `RD` correction gives a directory (§7 `RD`). |
| `new_file_name` | `string max=120` | Same kind as `file_name`; the corrected name an `RF` correction gives a file or alias (§7 `RF`). |
| `hash` | `string max=32` | Encoded content hash: exactly 32 lowercase hex characters for MD5 (§5 "Content hash encoding"). |
| `extra` | `string max=256` | Free-form field on `D`/`F` records (§7 `D`, §7 `F`), e.g. tags. Free-form within §4 "Field content": it may not contain `\|` or control characters, so an application storing structured data here must pick its own inner separator (e.g. `,` as in `tag1,tag2`). Left empty by any application that has no use for it. |
| `volume_id` | `uint32` | Unique within the library, assigned monotonically increasing, and never reused. |
| `target_volume_id` | `uint32` | Same kind as `volume_id`; the volume of the `F` record an `S` alias reaches for (§7 `S`). |
| `volume_label` | `string max=120` | Human/physical label printed on the medium. |
| `date_time` | `string max=15` | `YYYY-MM-DD_hhmm` — always when the *producing application* generated the surrounding index record(s), never the physical medium's own write timing (out of scope, §1). Producing one volume's index entries is treated as a single, fast step, so every `date_time` within that volume's section (`VOLUME_START`, `VOLUME_END`, any `AF`/`XF`/`XD`/`RD`/`RF`) conventionally carries the *same* value — it is not a start/end pair for how long the physical burn took. Mandatory on `VOLUME_START`/`VOLUME_END`, and optional but recommended on `AF`/`XF`/`XD`/`RD`/`RF` (§7 preamble) — recommended because these timestamps help reconstruct the history of changes to a specific file or directory, optional because that same date can otherwise be inferred from the enclosing volume section. |
| `total_vol_files` | `uint64` | Number of `F` records physically written in a given volume's own section (§7 `VOLUME_END`). |
| `vol_files_size` | `uint64` | Same kind as `size`; sum of `size` across that same volume's own `F` records — the size counterpart to `volume_hash` below (§7 `VOLUME_END`). |
| `volume_hash` | `string max=32` | Same shape as `hash` (§6 `hash`); a per-volume checksum derived from other hashes already in the index rather than from a file's own content (§7 `VOLUME_END`). |
| `file_key` | `volume_id file_id` | Composite: the full physical identity of a file record (§3). |

`file_key` is a documentation-level composite — it names the concept of "the
tuple that uniquely locates this record"; the `AF`/`XF`/`RF` `! RECORD`
declarations still spell out `volume_id`/`file_id` as separate fields rather
than referencing this composite name directly. Directories have no
equivalent composite: `dir_id` alone is already a directory's full
identity, since it's global across the library.

## 7. Record schema (`! RECORD`)

The ten record tags fall into two kinds. Five are **corrections** — `AF`,
`XF`, `XD`, `RD`, `RF` — they introduce nothing new; each acts on a record
already written by one of the other five, marking it canonical, gone, or
renamed/relocated. Those other five introduce **new data** the library
hasn't stated before — `VOLUME_START`/`VOLUME_END`, `D`, `F`, `S` — a new
volume section, directory, file, or alias.

A field name ending in `?` in a `! RECORD` declaration marks it **optional
and trailing**: a data line using that tag may omit it, having one fewer
pipe-delimited field than declared. As of `! VERSION 2.0` this applies to
`extra?` on `D` and `F` (below), and to `date_time?` on `AF`/`XF`/`XD`/`RD`/`RF`
(below) — every other field is mandatory. A conforming parser must accept a
line with or without its optional trailing field, treating a missing one as
empty/absent. This is what lets `extra` be added to a line later (e.g. to
attach tags), or `date_time` be left off a correction whose date is already
recoverable from its enclosing volume section (§6 `date_time`), without
invalidating older lines that predate the field, and without requiring every
existing line to be rewritten.

### `VOLUME_START` / `VOLUME_END` — volume boundary

```
! RECORD VOLUME_START volume_id volume_label date_time
! RECORD VOLUME_END volume_id volume_label date_time total_vol_files vol_files_size volume_hash last_dir_id
```

`VOLUME_START` declares one volume (one physical medium) and opens its
section; `VOLUME_END` closes it. Every `AF`, `XF`, `XD`, `RD`, `RF`, `D`,
`F`, and `S` record between a `VOLUME_START` and its matching `VOLUME_END`
was written as part of producing that volume, in whatever order the
archivist actually produced them (§8) — corrections are not required to be
grouped before new data:

- `AF`/`XF`/`RF` records may appear here even though they can reference a
  file on any *earlier* volume, or even a record introduced earlier in
  this very same section (§7 `AF`, `XF`, `RF`; §9 point 2) — their own
  explicit `volume_id` field is what matters for them, not their position
  in the file. `XD`/`RD` records likewise may reference a `dir_id`
  assigned by any earlier volume, or by this same section (§7 `XD`, `RD`)
  — directories need no `volume_id` at all, since `dir_id` is already
  global (§3, §10).
- `D` records in this range declare any directories newly introduced by
  this volume (§7 `D`) — a directory that already exists in the library
  isn't repeated.
- `F` records in this range derive their `volume_id` from this section (§7
  `F`) — this is the only place a file's volume is recorded.
- `S` records in this range derive their `volume_id` from this section the
  same way `F` does (§7 `S`) — though the *content* they reach for may
  live on any volume, via their own explicit `target_volume_id`.

Fields shared by both records:

- `volume_id` — internal numeric id of the volume within the library;
  repeated on `VOLUME_END` so a reader (or a human skimming the file) can
  confirm which section is closing, and so a truncated or corrupted section
  (a `VOLUME_START` with no matching `VOLUME_END`) can be detected. Volume
  sections MUST appear in strictly increasing `volume_id` order.
- `volume_label` — the label written/printed on the physical medium;
  repeated on `VOLUME_END` for the same corruption-detection reason as
  `volume_id` — if the two ends of a section ever disagree, something is
  wrong.
- `date_time` — when the producing application generated this volume's index
  entries (§6 `date_time`). Producing one volume's section is treated as a
  single, fast step — distinct from, and typically much shorter than, the
  physical burn — so this carries the *same* value on `VOLUME_START`,
  `VOLUME_END`, and any `AF`/`XF`/`XD`/`RD`/`RF` records the same section
  contains; it is not a start/end pair for the physical burn's duration.

A volume section without a valid matching `VOLUME_END` is incomplete. An
incomplete volume MUST NOT be considered part of the reconstructed library
state.

`VOLUME_END` MUST validate against its opening `VOLUME_START`:

- `VOLUME_END.volume_id` MUST equal the corresponding `VOLUME_START.volume_id`.
- `VOLUME_END.volume_label` MUST equal the corresponding
  `VOLUME_START.volume_label`.
- `VOLUME_END.date_time` MUST equal the corresponding `VOLUME_START.date_time`.

A reader applies a section's records to a working copy of the library's
state as it reads them, and commits that working state into the library's
durable state only once this validation succeeds — see §10 for the full
reading algorithm.

`VOLUME_END`-only fields — everything this volume itself physically
contributed, plus `last_dir_id` as the one whole-library value cheap
enough to keep without resolving any correction (§10):

- `total_vol_files` — count of `F` records physically written in this
  volume's own section. `S` records (§7 `S`) are never counted — an alias
  adds a new name, not new content.
- `vol_files_size` — sum of `size` across those same `F` records: this
  volume's own physical byte total, the size counterpart to `volume_hash`
  below.
- `volume_hash` — a checksum of exactly this volume's own `F` records,
  derived purely from their `hash` values already recorded a few lines up
  in this same section (§6 `hash`) — never by re-reading any file's actual
  content. Computed by concatenating, in the order they appear in this
  volume's section, each `F` record's `hash` field — with no separator
  between them, since every `hash` in a given file is the same fixed
  length (§6 `hash`) for as long as `! CONTENT_HASH_ALGORITHM`/
  `! CONTENT_HASH_ENCODING` stay unchanged, which makes plain
  concatenation unambiguous — then applying `! CONTENT_HASH_ALGORITHM` to
  the resulting text, taken as its ASCII bytes (i.e. the lowercase hex
  characters themselves, not the digest bytes they encode), and encoding
  the result with `! CONTENT_HASH_ENCODING` (§5) — the same two header
  declarations already used for every per-file hash, just applied over a
  different input. `S` records contribute nothing, since they carry no
  `hash` of their own (§7 `S`); a volume that happens to add no `F` records
  at all uses the hash of an empty input (`d41d8cd98f00b204e9800998ecf8427e`
  for MD5). Because the input is plain text, this too can be reproduced
  with stock OS tools, e.g. `printf '%s' HASH1 HASH2 ... | md5sum` on
  Linux or `printf '%s' HASH1 HASH2 ... | md5` on macOS.
- `last_dir_id` — highest `dir_id` assigned in the library as of this
  volume; a simple monotonic maximum, unaffected by
  `AF`/`XF`/`XD`/`RD`/`RF` (§7 `D`) — the one whole-library value
  `VOLUME_END` still carries, since no correction history needs resolving
  to know it.

`total_vol_files`/`vol_files_size` are the only per-volume tallies
`! VERSION 2.0` keeps — `S` additions, and every correction (`AF`, `XF`,
`XD`, `RD`, `RF`), have no dedicated counter of their own. Neither field
says anything about the library's *active* file count or byte size once
later corrections are taken into account (§10) — a reader wanting that
resolves the full cumulative log, the same way it resolves anything else
in the active view; a reader wanting to know how many corrections a given
volume's section contains counts the `AF`/`XF`/`XD`/`RD`/`RF` lines in that
section directly, the same way it would count anything else in the file.

In practice, these two fields (together with `volume_hash`) exist to make a
volume's physical medium independently checkable: re-reading that volume's
`F` records and recomputing their `size`/`hash` lets a verifier confirm the
medium still matches what the index recorded, without resolving any
correction history first (§10 has the full recipe).

```
VOLUME_START|2|Music-002|2026-08-14_1111
XF|1|7|2026-08-14_1111
...
VOLUME_END|2|Music-002|2026-08-14_1111|6|41474037|3205bef3b9444985cee61145b49a55df|5
```

### `D` — Directory

```
! RECORD D dir_id parent_dir_id dir_name extra?
```

A logical directory in the library's namespace tree.

- `dir_id` — global across the whole library: assigned once, the first time
  a volume introduces this directory, and never reused or re-assigned by a
  later volume — even one that adds more files under it. `0` always denotes
  the library's root — it has no name or parent of its own, so it needs no
  `D` record; every actual directory is numbered from `1` upward, in the
  order it's introduced, wherever a `dir_id`-shaped field references it
  (`parent_dir_id` below, and `F`/`S`'s own `dir_id`, §7 `F`, §7 `S`).
- `parent_dir_id` — the parent directory's `dir_id` (`0` = root). Because
  `dir_id` is global, the parent may well have been declared by an earlier
  volume, not the current one.
- `dir_name` — directory name.
- `extra?` — optional, trailing (§7 preamble); free-form (§6 `extra`), e.g.
  tags. Empty or omitted unless an application populates it.

Directories form a single, stable, append-only namespace tree for the whole
library (not one tree per volume). Empty directories may be omitted.
Because `dir_id` is global, a given logical folder (e.g. `Band - Album`) is
declared exactly once, by whichever volume first introduces it — a later
volume that adds files under an already-existing folder does not re-declare
it, it simply references the existing `dir_id` from its `F` records (see the
worked example, §9).

**A `(parent_dir_id, dir_name)` pair is permanently unique.** Two directories
may share a `dir_name` (with different parents) or share a `parent_dir_id`
(with different names), but never both at once — no two directories may ever
simultaneously carry the same `parent_dir_id` and the same `dir_name`. This
holds across the directory's entire history, not just its currently active
content: even a directory whose entire subtree has since been marked gone
via `XD` keeps occupying its `(parent_dir_id, dir_name)` pair, since `XD` doesn't
delete the directory itself (§7 `XD`). Continuing or recovering the *same*
logical folder therefore reuses its existing `dir_id` rather than minting a
new one under the same name and parent (see §9 for a worked recovery
example); a genuinely new directory that would otherwise collide with an
existing name is resolved with `RD` instead (see the `(flac)`/`(mp3)` split,
§9 point 5).

```
D|1|0|Band
D|2|1|Band - Album
```

### `F` — File

```
! RECORD F file_id dir_id file_name size hash extra?
```

An archived file record. **`volume_id` is not a field of `F`.** A file's
volume is positional, not carried on the record itself: an `F` record
belongs to whichever volume's `VOLUME_START`...`VOLUME_END` section it
physically appears inside (§7 `VOLUME_START`/`VOLUME_END`) — a reader tracks
"current volume" while scanning sequentially and stamps it onto each `F`
line encountered.

This is a deliberate space-saving choice: real libraries will have vastly
more files than volumes, so repeating `volume_id` on every `F` record would
be wasteful, while tracking it once per section costs a reader nothing
extra. The file's *directory*, by contrast, is not positional: `dir_id` is
global across the library (§7 `D`), so a reader resolves it against the
single library-wide directory tree built from all `D` records seen so far,
regardless of which volume declared that directory.

- `file_id` — unique only within `volume_id` (full physical identity is
  `(volume_id, file_id)`, i.e. `file_key`); `volume_id` itself comes from
  the enclosing volume section, not from this record.
- `dir_id` — the containing directory's global id (§7 `D`); `0` means the
  file lives directly in the library's root, the same sentinel
  `parent_dir_id` uses (§7 `D`) — the root has no `D` record to reference
  instead.
- `file_name` — file name (not identity, see §3).
- `size`, `hash` — content identity (§3); used to detect duplicates already
  present anywhere else in the library and to verify integrity.
- `extra?` — optional, trailing (§7 preamble); free-form (§6 `extra`), e.g.
  tags. Empty or omitted unless an application populates it.

```
F|1|2|cover.jpg|125142|7fb5d46fb9bff11f638d42c231936c83
F|2|2|...01 Track One.flac|11124049|ef4e7f1fe605a39cd47f2a05f58faee5|tag1,tag2
```

### `S` — Symlink

```
! RECORD S file_id dir_id file_name target_volume_id target_file_id
```

An alias: a new tree position that reaches for content an `F` record
elsewhere already holds, instead of archiving another physical copy of it.
`S` exists for libraries that try to avoid storing duplicate physical
copies of the same content (see `AF` below): without it, making the same
content reachable from more than one place in the tree (e.g. the same
`cover.jpg` filed under two different album folders) would mean either
writing those bytes to the volume a second time, or not having the second
place at all.

- `file_id` — this alias's own local id, drawn from the *same* per-volume
  counter as `F`'s `file_id`, not a separate numbering space — so
  `(volume_id, file_id)`, with `volume_id` derived positionally exactly
  like `F`'s (§7 `VOLUME_START`/`VOLUME_END`), addresses either an `F` or
  an `S` uniformly. This is what lets `XF`/`RF` correct an alias the same
  way they correct a file (§7 `XF`, §7 `RF`).
- `dir_id` — where this alias lives in the library's tree (global, exactly
  like `F`'s `dir_id`, including `0` meaning the root, §7 `D`, §7 `F`).
  `AF`/`XF` address an existing record purely by `file_key` and need no
  `dir_id` at all; `S` is different because it creates a brand-new tree
  position, so it needs this field to say where that position is.
- `file_name` — this alias's own name at that position; independent of the
  target's own `file_name` (§7 `F`) — an alias may sit under the same name as
  its target or under a completely different one.
- `target_volume_id`, `target_file_id` — the existing `F` record whose
  content this alias reaches for (can be on any volume, same pattern as
  `AF`/`XF`, §3 `file_key`).

The immediate target of an `S` MUST identify an existing `F` record. An `S`
MUST NOT target another `S`.

`S` carries no `size`/`hash` of its own — the content it reaches for is
already accounted for by its target's `F` record, so an alias adds a new
name to the tree without adding new bytes or new counted content (§7
`VOLUME_END`, §10).

**Resolving a target through `AF`/`XF`/`XD`.** An alias's target field
identifies *which content* it reaches for, not a permanently fixed
physical record: a reader resolves it the same way it resolves any other
content in the library, through whatever `AF`/`XF`/`XD` history applies to
that content (§10 "Resolving the active view"). Concretely: if the target
record is later superseded by an `AF` (a genuine second physical copy of
that content is named canonical instead — see below), the alias
transparently follows to the new canonical record; if the target record is
later marked `XF` (or its containing directory is removed via `XD`) and no
other copy of that content survives elsewhere in the library, the alias is
dangling, exactly as it would be if the content were simply gone. `RF`/`RD`
corrections to the target's own name/location don't affect the alias at
all — it addresses the target by `file_key`, not by `file_name`, so the
target can be renamed or relocated freely without touching anything that
aliases it.

```
S|1|6|cover.jpg|1|1
```

(From `Music-003`'s section in `samples/music.bdea` — creates an alias with
its own `file_id=1`, filed under `dir_id=6`, named `cover.jpg`, that reaches
for the `F` record `file_id=1` on `volume_id=1`; see §9 for the full
context.)

### `AF` — Canonical (Active) File

```
! RECORD AF volume_id file_id date_time?
```

Declares that `(volume_id, file_id)` is the canonical/preferred physical
record to use for its content going forward. Like `S` (§7 `S`), `AF`
exists for libraries that try to avoid ending up with duplicate physical
copies of the same content: without it, re-archiving a file whose content
is already in the library would simply leave two independent, equally
"active" copies. With `AF`, the archivist settles which physical copy is
the current one; a symlink to it, if one is needed elsewhere in the tree,
is then made with `S`. Both `AF` and `S` are optional — a library that
doesn't care about deduplication can simply not use either.

Unlike `D`/`F`/`S`, `AF` can point at a file on **any** volume (not just
the current one). Typical use: the same content ends up archived more than
once — possibly across several different volumes (matching `size`+`hash`,
e.g. the same file re-added later on a different volume, per §9) — `AF`
names exactly one of those records as "the" file to present/use going
forward. This does **not** require the other, superseded record(s) to be
marked `XF` — they can remain perfectly valid, readable archived copies;
they're simply no longer the one pointed to as canonical. However many
copies of that content exist archive-wide, exactly one stays counted in
the library's active view once `AF` names it — the rest drop out of that
count without needing `XF`.

- `volume_id`, `file_id` — the record being named canonical (§3 `file_key`).
  Always an `F` record, never an `S` (§7 `S`) — an alias holds no content
  of its own, so it can never be *the* canonical physical copy of
  anything; canonical-for-content only makes sense among records that
  actually hold bytes.
- `date_time?` — optional but recommended (§6 `date_time`): when this
  correction was appended to the index — not a property of either physical
  copy. If present, this **is** that timestamp; if absent, it's taken from
  the enclosing volume's `VOLUME_START`/`VOLUME_END` timestamp instead,
  since a correction's date always coincides with its enclosing volume's
  date anyway.

`AF` does not invalidate, delete, or otherwise remove any other `F` record.
It only selects one surviving `F` record as the canonical representation of
the content. At most one `F` record SHOULD be canonical for a given
`size`+`hash` content at any point in the active view.

If more than one `AF` is issued for the same content over time, each `AF`
updates which record is currently canonical for that content — a later
`AF` supersedes an earlier one, the same way `RD`/`RF` resolve for a
directory or file (§4, §10).

An `AF`'s target need not come from an earlier volume — it may name a
record introduced earlier in this very same section canonical, e.g. right
after the `F` record that just wrote it, when the archivist already knows
at write time which copy should be preferred (§9 point 2). `AF` is a
correction like any other: the only requirement is that its target already
exist earlier in the log (§4, §8), not that it be grouped with other
corrections or deferred to a later volume.

If the record an `AF` names canonical is later marked `XF` (or falls under
an `XD`), that `AF` is not automatically inherited by any other surviving
copy of the same content — the content simply has no canonical record from
that point on, until a new `AF` names one. Recovering from the loss of a
canonical copy (e.g. re-archiving the content fresh on a later volume, §9
point 6) means issuing a fresh `AF` for the new record, exactly like any
other case of one `AF` superseding an earlier one.

```
AF|2|6|2026-08-14_1111
```

(From `Music-002`'s section in `samples/music.bdea` — names `volume_id=2,
file_id=6` canonical immediately after that very record's own `F` line, in
the same section: the archivist already knows this fresh copy should be
preferred over `Music-001`'s original, which had turned out to read
unreliably; see §9 for the full context.)

```
AF|4|6|2026-08-24_1000
```

(From `Music-004`'s section in `samples/music.bdea` — `Music-002`'s
canonical copy was marked `XF` earlier in this same section, since that
whole medium failed; this `AF`, placed right after the `F` record that
re-archives the content fresh, names that replacement canonical instead.
See §9 point 6.)

### `XF` — Delete File

```
! RECORD XF volume_id file_id date_time?
```

Declares that the record at `(volume_id, file_id)` must be skipped when
reconstructing the library from that point on. Like `AF`, it can reference
any volume's record. What "gone" means depends on what kind of record it
is:

- For an `F` record: the physical copy has become unavailable —
  unreadable, damaged, or lost from that specific medium. If no other,
  unspoiled `F` record with the same `size`+`hash` exists elsewhere in the
  library, that content is effectively lost.
- For an `S` record (§7 `S`): the alias itself has been removed — nothing
  about the underlying content changes, it simply stops being reachable
  from that one tree position.

- `volume_id`, `file_id` — the record being marked gone (§3 `file_key`).
- `date_time?` — optional but recommended (§6 `date_time`): when this
  correction was appended to the index — not when the medium actually
  failed or was noticed as bad. If present, this **is** that timestamp; if
  absent, it's taken from the enclosing volume's `VOLUME_START`/
  `VOLUME_END` timestamp instead.

```
XF|1|7|2026-08-14_1111
```

(From `Music-002`'s section in `samples/music.bdea` — forcibly excludes
`volume_id=1, file_id=7`, a track mistakenly archived onto that medium,
from the library; see §9 for the full context.)

### `XD` — Delete Directory

```
! RECORD XD dir_id date_time?
```

A shorthand for `XF`: declares every `F`/`S` record filed anywhere under
`dir_id` — including in any descendant directory, regardless of which
volume(s) they were written on — gone at once, instead of writing one `XF`
per file. `XD` exists purely for this convenience and carries no purpose
beyond it: if it didn't exist, marking each of those records `XF`
individually would produce exactly the same active view. Written when a
whole branch is gone at once, e.g. the surviving, still-readable part of a
damaged volume didn't include this branch.

- `dir_id` — the directory whose subtree is marked gone this way (§7 `D`).
  Because `dir_id` is never reused (§7 `D`), this alone is enough to
  identify the target — no companion field is needed the way `XF` needs
  `volume_id` alongside `file_id`.
- `date_time?` — optional but recommended (§6 `date_time`): when this
  correction was appended to the index. If present, this is that
  timestamp, exactly like `XF`; if absent, it's taken from the enclosing
  volume's `VOLUME_START`/`VOLUME_END` timestamp instead.

`XD` does not delete the directory itself — `dir_id` remains a permanent
logical path in the tree (§7 `D`), and its own `D` record (and any
descendant `D` records) stays exactly as declared. What `XD` removes is
content: every `F`/`S` record filed under `dir_id` at the time of this
correction. A later volume can still file new `F`/`D` records under the
same `dir_id`, exactly as it could before the `XD` — the directory keeps
existing as a path; only what was filed in it at the time of the
correction is gone.

**Ordering within the same section.** Because `(parent_dir_id, dir_name)` stays
permanently claimed by a directory even after `XD` (§7 `D`), recovering the
same logical folder — e.g. right after the volume holding its previous
copies fails — reuses that directory's existing `dir_id` rather than minting
a new one. If the *same* volume both retires a `dir_id` this way and re-files
new `F`/`D` records under it, the `XD` must appear *before* those new
records in that volume's section: a reader processes the log in order (§4, §10),
and `XD` wipes whatever is filed under `dir_id` at its own position in the
log — records placed ahead of it would be wiped by the very `XD` meant to
close out the old ones (see §9 for a worked recovery example). This is one
of only two hard ordering constraints the format imposes on a section's
records at all (§8) — everything else about how corrections and new data
interleave is convention, not a requirement.

Like `XF`, this is a statement about the index's active view, not a claim
about the medium's material condition — the format doesn't distinguish a
truly unrecoverable copy from one an archivist simply chose to treat as
gone.

```
XD|4|2026-08-24_1000
```

(From `Music-004`'s section in `samples/music.bdea` — drops `dir_id=4` and
the five `F` records filed under it from the active view at once, instead
of writing one `XF` per file; see §9 for the full context.)

### `RD` — Rename/Relocate Directory

```
! RECORD RD dir_id new_parent_dir_id new_dir_name date_time?
```

Corrects an existing directory's `parent_dir_id` and/or `dir_name` (§7 `D`)
going forward, without writing any new `D`, `F`, or `S` record and without
touching any archived bytes. Because the tree is built from
`dir_id`/`parent_dir_id` links rather than names (§7 `D`), one `RD` moves
or renames the entire subtree beneath `dir_id` at once — every file and
subdirectory under it keeps referencing the same `dir_id`, so none of them
needs correcting individually.

- `dir_id` — the directory being corrected. Unchanged: `dir_id` is the
  directory's permanent identity (§7 `D`); only its position/label in the
  tree moves.
- `new_parent_dir_id` — the corrected parent (§6 `new_parent_dir_id`);
  repeat the existing `parent_dir_id` unchanged if this correction is a
  pure rename.
- `new_dir_name` — the corrected name (§6 `new_dir_name`); repeat the
  existing `dir_name` unchanged if this correction is a pure move.
- `date_time?` — optional but recommended (§6 `date_time`): when this
  correction was appended to the index. If present, this is that
  timestamp, exactly like `XF`; if absent, it's taken from the enclosing
  volume's `VOLUME_START`/`VOLUME_END` timestamp instead.

If more than one `RD` targets the same `dir_id` over time, each `RD`
updates the directory's current `parent_dir_id`/`dir_name` in turn — a
later `RD` supersedes an earlier one. A reader applies each `RD` to its
working state as it is encountered, so the directory's current
`parent_dir_id`/`dir_name` is simply whatever that state holds after the
last `RD` applied so far, or its original `D` record if none has been
applied yet (§4, §10).

```
RD|2|1|Artist One - Album One (flac)|2026-08-20_0900
```

(From `Music-003`'s section in `samples/music.bdea` — a new sibling folder,
`D|6|1|Artist One - Album One (mp3)`, is about to be
introduced under `Artist One` (`dir_id=1`) later in this same section, so
this `RD` renames the existing flac folder, `dir_id=2`, ahead of it to add
the `(flac)` suffix — parent stays `1`, only `dir_name` changes, since the
two are about to need distinguishing names. Every `F`/`S` record already filed
under `2` moves with it without being touched; see §9.)

### `RF` — Rename/Relocate File

```
! RECORD RF volume_id file_id new_dir_id new_file_name date_time?
```

Corrects an existing `F` or `S` record's `dir_id` and/or `file_name` going
forward, without writing its content again. This is the mechanism for a
pure naming or filing mistake: the record's `size`+`hash` (for an `F`) or
target (for an `S`) never change, only where it sits in the tree and what
it's called there.

- `volume_id`, `file_id` — the record being corrected (§3 `file_key`);
  addresses either an `F` or an `S`, the same way `AF`/`XF` do (§7 `S`).
- `new_dir_id` — the corrected directory (§6 `new_dir_id`); repeat the
  existing `dir_id` unchanged if this correction is a pure rename.
- `new_file_name` — the corrected name (§6 `new_file_name`); repeat the
  existing `file_name` unchanged if this correction is a pure move.
- `date_time?` — optional but recommended (§6 `date_time`): when this
  correction was appended to the index. If present, this is that
  timestamp, exactly like `XF`; if absent, it's taken from the enclosing
  volume's `VOLUME_START`/`VOLUME_END` timestamp instead.

If more than one `RF` targets the same `(volume_id, file_id)` over time,
each `RF` updates that record's current `dir_id`/`file_name` in turn — a
later `RF` supersedes an earlier one, applied the same way `RD` resolves
for a `dir_id` (§4, §10). `RF` is unrelated to `AF`: `AF` picks which of several independent physical
copies of the *same content* is canonical; `RF` corrects where *one
specific* record sits in the tree, whether or not any duplicate of it
exists.

```
RF|1|3|2|Artist One - Album One - 02 Track Two.flac|2026-08-20_0900
```

(From `Music-003`'s section in `samples/music.bdea` — fixes `Music-001`'s
naming mistake on `F|3|2`, dropping a stray "label" from `...02 Track Two
label.flac`; `dir_id` stays `2`, only `file_name` changes. A pure naming
fix, no duplicate involved; see §9.)

`AF`, `XF`, `XD`, `RD`, and `RF` are independent mechanisms — a given
correction may use any combination of them, but none implies another.
Together they're how BDEA supports corrections without rewriting
write-once media: an old record is left physically in place; `AF` redirects
which copy is treated as canonical, `XF`/`XD` flag a copy (or a whole
branch) as gone, and `RD`/`RF` fix a naming or filing mistake in place.

All five are manual, archivist-issued corrections — nothing in the format
writes them automatically. `AF`/`S`/`XF` are the responses to a genuine
same-`size`+`hash` match that an application surfaces (§10): name one copy
canonical if both stay archived (write `AF`), add an alias instead of
writing the bytes again if the content deliberately belongs in more than
one place (write `S`, §7 `S`), record that a copy has gone bad (write
`XF`, or `XD` if a whole directory went with it), or simply leave two
independent files that happen to share content alone. A pure naming or
filing mistake — no real duplicate involved — is corrected with `RD`/`RF`
instead, which needs no new bytes and has nothing to do with duplicate
detection.

## 8. Volume sections and growth over time

A file's body is a sequence of volume sections in the order they were
produced:

```
VOLUME_START|1|Music-001|2026-08-13_1232

D|1|0|Artist One
D|2|1|Artist One - Album One
...
F|1|2|cover.jpg|125142|...

VOLUME_END|1|Music-001|2026-08-13_1232|8|82235912|d83d681ff83f05ceb2348792b9300a01|3

VOLUME_START|2|Music-002|2026-08-14_1111

XF|1|7|2026-08-14_1111

D|4|0|Artist Two
...
F|1|5|...

VOLUME_END|2|Music-002|2026-08-14_1111|6|41474037|3205bef3b9444985cee61145b49a55df|5
```

Each new volume appends: one `VOLUME_START` line; then, in whatever order
the archivist actually produced them, any `AF`/`XF`/`XD`/`RD`/`RF`
corrections (which may reference files or directories from an earlier
volume, or from earlier in this very same section) together with any new
`D` records for directories this volume introduces (directories that
already exist are not repeated), `F` records, and `S` aliases to content
already archived; and closes with a matching `VOLUME_END` line.

A section's records are processed strictly in the order they appear (§4)
— there is no requirement to group every correction ahead of every
new-data record. In practice corrections usually do come first, since most
mistakes concern content from an already-closed earlier volume, discovered
only after this volume's own new content is already decided (most of §9's
volumes follow that pattern). But nothing stops a correction from
following the very record it concerns, in the same section instead — e.g.
naming a freshly-written duplicate canonical with an `AF` right after its
own `F` record, rather than waiting for a later volume (§9 point 2; §7
`AF`).

The format places exactly two hard constraints on this ordering, both
already implied by records being processed strictly in order (§4):

- A record can only reference something that already exists earlier in
  the log — from an earlier volume, or from earlier in this same section.
  A correction can't name a target that hasn't been written yet.
- `XD` (and, one file at a time, `XF`) removes whatever is filed under its
  target *as of its own position* in the log (§7 `XD`). If the same
  volume both retires content this way and re-files new content under the
  same `dir_id`, the correction must come *before* that new content —
  otherwise the very correction meant to clear the old content would wipe
  the new content out too.

**Important consequence of write-once media:** because a volume, once
physically written, can't be rewritten, its embedded copy of the index is
frozen at whatever the library looked like when that volume was produced.
Corrections made later only appear in the index copies written onto
*subsequent* volumes (and in the working master copy on the computer
building the archive) — an older volume's own copy will not show later
`AF`/`XF`/`XD`/`RD`/`RF` corrections. When reconciling state, prefer the copy
from the highest-numbered (most recent) volume available.

## 9. Worked example

Walking through [`samples/music.bdea`](samples/music.bdea), across its four
volumes:

1. **Music-001** is burned first. It declares `Artist One` (`dir_id=1`)
   directly under the library root, and under it the album folder `Artist
   One - Album One` (`dir_id=2`) plus an unrelated sibling folder, `.info`
   (`dir_id=3`). `dir_id=2` gets a cover image and five flac tracks
   (`file_id` 1–6), each flac track's `F` record carrying `tag1,tag2` in
   its trailing `extra` field (§6, §7 `F`) — a concrete illustration of
   `extra` being free-form; nothing else in the sample interprets these
   tags, they're just carried along. One of them, `F|4|2|...03 Track
   Three.flac`, turns out — once this volume is actually read back — to be
   unreliable: it comes back correctly only some of the time, not
   consistently unreadable, just not trustworthy. That's what prompts a
   fresh safety copy of the same content on the very next volume, one
   already known at write time to deserve becoming canonical (`Music-002`,
   point 2). Another of them, `F|3|2|...02 Track Two
   label.flac`, is filed with a naming mistake — a stray "label" in the
   name — corrected later in `Music-003` (point 5). One more file in the
   same folder, `F|7|2|Other Artist - Stray Track.flac`, is unrelated
   content that got mistakenly archived there too; it's forcibly excluded
   from the library in `Music-002` (point 3) — not a case of the medium
   itself going bad, but of catching a stray extraneous file after the
   fact. `dir_id=3` (`.info`) holds one small non-music file,
   `F|8|3|filedata.txt`; that file (not the folder itself) is dropped from
   the active view in `Music-002` as well. Closing this volume's section,
   `VOLUME_END|1|Music-001|2026-08-13_1232|8|82235912|d83d681ff83f05ceb2348792b9300a01|3`
   reports `8` (`total_vol_files`) and `82235912` (`vol_files_size`) — this
   volume's own `F` count and combined byte size, simple sums over the `8`
   `F` records just written, since no correction has touched anything yet.
   Its `volume_hash`, `d83d681ff83f05ceb2348792b9300a01`, concatenates those same
   `8` records' own `hash` values, in the order they appear (no separator
   needed, since they're all the same fixed length), then hashes and
   encodes that text the same way `CONTENT_HASH_ALGORITHM`/
   `CONTENT_HASH_ENCODING` (§5) already hash file content. No archived byte
   is touched to produce it — it's built purely from hashes already
   sitting in the index. `last_dir_id` is `3`, the highest `dir_id` this
   volume assigned.
2. **Music-002** is burned later. It declares the one genuinely new folder
   tree it needs, `Artist Two` (`dir_id=4`) and her album `Album Two
   (2018)` (`dir_id=5`), and adds that artist's five mp3 tracks there
   (`file_id` 1–5). It also re-archives, as a safety copy, the "03 Track
   Three" content whose original (`Music-001`'s `F|4|2`) had turned out
   unreliable (point 1):
   `F|6|2|...03 Track Three.flac|11707500|ef106934d3c6396b338552a0c2396d35|tag1,tag2`
   — `size`+`hash` **identical** to `Music-001`'s `F|4|2`, the same
   content-identity match the format's model (§3) exists to catch. But
   here the archivist already knows, at the moment this line is written,
   *why* the copy was made and which one should be preferred going
   forward — so the very next line in this same section is
   `AF|2|6|2026-08-14_1111`, naming this fresh copy canonical immediately,
   instead of waiting for a later volume the way a correction normally
   would (§7 `AF`; §8). `Music-001`'s original isn't touched or marked
   `XF` — it remains a valid, readable archived copy, just no longer the
   presented/canonical one.
3. At the start of the `Music-002` section, right after `VOLUME_START`,
   two unrelated corrections are recorded together, ahead of any new data
   — the conventional placement, since both concern mistakes already
   found in `Music-001`'s closed-out history, unrelated to whatever this
   volume is about to add:
   - `XF|1|7|2026-08-14_1111` — declares that `Music-001`'s file 7
     (`Other Artist - Stray Track.flac`) is forcibly excluded from the
     library. This isn't the case of pulling a volume back out and finding
     part of it unreadable — it's the opposite: after burning a large
     volume, this unrelated track turned out to have been mistakenly
     included on it, and the next volume's section is where that mistake
     gets corrected. Since nothing else in the library shares its hash,
     that content is simply gone from the active library unless someone
     deliberately re-adds it later.
   - `XD|3|2026-08-14_1111` — drops the one file filed under `Music-001`'s
     `.info` folder, `F|8|3|filedata.txt` (`dir_id=3`), from the active
     view. The folder itself is not removed: `dir_id=3` stays a permanent,
     valid logical path (§7 `XD`) — it's just empty of active content until
     something new is filed under it.

   The section's *third* correction, `AF|2|6|2026-08-14_1111` (point 2),
   breaks that front-loaded convention on purpose: it comes near the
   *end* of the section, right after the `F|6|2` record it names
   canonical, because its target simply didn't exist yet at the start of
   the section — an `AF` can only reference a record already earlier in
   the log (§4, §8), and here that record is this same section's own
   duplicate, not anything from an earlier volume. Nothing requires
   `AF`/`XF`/`XD`/`RD`/`RF` to cluster together at one spot in a section;
   each simply has to follow whatever it targets (§8).
4. Closing `Music-002`'s section,
   `VOLUME_END|2|Music-002|2026-08-14_1111|6|41474037|3205bef3b9444985cee61145b49a55df|5`
   reports `6` for `total_vol_files` and `41474037` for `vol_files_size` —
   this volume's own `6` `F` records (the `5` new mp3 tracks plus the
   safety-copy flac from point 2) and their combined `size`, counted
   exactly as physically written — `total_vol_files`/`vol_files_size` are
   per-volume physical facts, not the library's resolved active view, and
   `AF` never adds to either count (§7 `VOLUME_END`). `volume_hash`,
   `3205bef3b9444985cee61145b49a55df`, is built the same way as `Music-001`'s
   but from only this volume's `6` `F` hashes — it does not extend or
   combine with `Music-001`'s `volume_hash`, since `volume_hash` is always
   scoped to one volume, never a whole-library value. `last_dir_id`
   becomes `5`, the highest `dir_id` either volume has assigned so far. A
   reader wanting the library's *active* file count/size at this point
   resolves the full log instead (§10): of the `14` `F` records written so
   far (`8` from `Music-001` plus `6` from `Music-002`), `Music-001`'s
   `F|7|2` (`XF`'d) and `Music-001`'s `F|8|3` (`XD`'d) are inactive, and
   `Music-001`'s `F|4|2`/`Music-002`'s `F|6|2` — same content — already
   collapse into a single active record now that this section's own `AF`
   has run (point 2), leaving `11` active records — but no field in
   either `VOLUME_END` states `11` directly.
5. **Music-003** is burned later still. Its section combines corrections
   (`RD`, `RF`) with genuinely new content of its own: one new directory,
   an alias, and three new mp3-format files — no `AF` here, since that
   correction already ran back in `Music-002` (point 2):
   - `RD|2|1|Artist One - Album One (flac)|2026-08-20_0900` (§7 `RD`)
     renames `dir_id=2` — parent unchanged at `1` — to add the "(flac)"
     suffix, ahead of the "(mp3)" sibling this same section is about to
     introduce below, so the two end up with distinguishing names.
   - `RF|1|3|2|Artist One - Album One - 02 Track Two.flac|2026-08-20_0900`
     (§7 `RF`) fixes `Music-001`'s `F|3|2` naming mistake from point 1,
     dropping the stray "label" word; `dir_id` stays `2`, unchanged — a
     pure naming fix, no duplicate involved.
   - `D|6|1|Artist One - Album One (mp3)` introduces a new sibling folder
     directly under `Artist One` (`dir_id=1`), alongside the existing flac
     album folder (`dir_id=2`, just renamed above).
   - `S|1|6|cover.jpg|1|1` aliases `Music-001`'s `F|1|2|cover.jpg` into the
     new mp3 folder instead of archiving those bytes a second time (§7
     `S`).
   - `F|2|6`, `F|3|6`, `F|4|6` add mp3 versions of three of the flac
     tracks — different bytes and `hash`, not duplicates of their flac
     counterparts despite sharing a piece of content conceptually.
   - Closing this volume,
     `VOLUME_END|3|Music-003|2026-08-20_0900|3|11577606|9d9d004ac598249931471f7ce8f20c46|6`
     reports `3` new `F` records (`RD`/`RF`/`S` add no `F` of their own)
     and `11577606` for their combined size. `last_dir_id` becomes `6`,
     the one new `D` this volume introduced. `volume_hash`
     (`9d9d004ac598249931471f7ce8f20c46`) is built from only these `3` hashes.
6. **Music-004** is burned after `Music-002`'s physical medium turns out to
   have failed entirely. Recovering from that means marking whatever lived
   solely on that medium as gone, then re-archiving the same content fresh
   — under the *same* directories, since a `(parent_dir_id, dir_name)` pair
   stays permanently claimed even once a directory's content is marked
   gone via `XD` (§7 `D`), so recovering a logical folder reuses its
   existing `dir_id` rather than minting a new one:
   - `XF|2|6|2026-08-24_1000` marks `Music-002`'s "03 Track Three" safety
     copy (`volume=2, file=6`) gone, since that whole medium is
     unreadable. Unlike a duplicate that was never canonical, this one
     matters: it's the very record `Music-002`'s own `AF` (point 2) named
     canonical, so this `XF` leaves that content with *no* canonical
     record at all until a fresh `AF` names one below (§7 `AF`) —
     `Music-001`'s original still exists and is still active, but nothing
     currently says it's preferred, and it's the one copy already known
     to read unreliably (point 1).
   - `XD|4|2026-08-24_1000` drops the five `F` records filed anywhere under
     the `Artist Two` branch (`dir_id=4` and its child `dir_id=5`) from the
     active view in one record, since none of that content survives
     anywhere else (§7 `XD`) — `dir_id=4`/`dir_id=5` themselves aren't
     removed, just left empty of active content. Both corrections appear
     *before* this volume's new `F` records below — not because
     corrections must always come first (point 2 already shows otherwise),
     but because `XD` specifically only wipes what's filed under its
     `dir_id` as of its own position in the log (§7 `XD`), so the
     recovered files need to come *after* it, not before, or this same
     `XD` would wipe them out too. `XF` carries no such forward-wipe
     effect, so `XF|2|6` could in principle sit anywhere in this section —
     it's placed first here simply because that's the order the recovery
     actually happened in.
   - `F|1|5` through `F|5|5` re-add the same five mp3 tracks — with the
     same `size`+`hash` as `Music-002`'s originals, genuinely the same
     content, recovered from the archivist's still-intact source files —
     filed under the *same* `dir_id=5` (`Artist Two - Album Two (2018)`)
     `Music-002` originally used.
   - `F|6|2|...03 Track Three.flac|11707500|ef106934d3c6396b338552a0c2396d35|tag1,tag2`
     re-archives the "03 Track Three" content once more — the same
     `size`+`hash` as both `Music-001`'s `F|4|2` and the now-dead
     `Music-002` `F|6|2` — filed under the same `dir_id=2`. None of
     `dir_id=4`, `dir_id=5`, or `dir_id=2` gets a new `D` record: all
     three `(parent_dir_id, dir_name)` pairs already belong to them
     permanently (§7 `D`) — `Music-004` introduces no new directory at
     all.
   - `AF|4|6|2026-08-24_1000` immediately names this fresh copy canonical
     — the same pattern as `Music-002`'s own `AF` (point 2), following its
     own section's `F` record rather than waiting for a later volume.
     This is what actually restores the content's canonical status after
     `XF|2|6` took it away above; without this line, the "03 Track Three"
     content would be left with no canonical record at all, even though
     three archived copies of it now exist across three different
     volumes.
   - Closing this volume,
     `VOLUME_END|4|Music-004|2026-08-24_1000|6|41474037|3205bef3b9444985cee61145b49a55df|6`
     reports `6` `F` records this volume physically wrote (the `5`
     recovered mp3 tracks plus the recovered flac) and `41474037` for
     their combined size — identical to `Music-002`'s own
     `total_vol_files`/`vol_files_size`, since `Music-004` recovers
     exactly the same six pieces of content `Music-002` originally wrote,
     in the same order. Its `volume_hash`, `3205bef3b9444985cee61145b49a55df`,
     comes out **identical** to `Music-002`'s own `volume_hash` for the
     same reason — not a coincidence or a bug: `volume_hash` is a pure
     function of the concatenated `hash` values in order (§7
     `VOLUME_END`), and both volumes here concatenate the exact same six
     hashes in the exact same order. `last_dir_id` stays `6` —
     `Music-004` assigns no new `dir_id` at all, since it only reuses
     existing ones. A reader resolving the active view after this volume
     lands on the same `14` active files and the same `100244460` total
     byte size as after `Music-003` (§10): the "03 Track Three" content
     has changed which physical copy is canonical twice now —
     `Music-001`'s original, then `Music-002`'s safety copy, then this
     volume's fresh recovery — but at every point it still resolves to
     exactly one active record, so none of that churn moves the library's
     overall totals.

## 10. Notes for implementers

These are consequences of the format worth keeping in mind when building
an application against it. They are advice, not additional format rules —
nothing here overrides the normative definitions given in the sections
above, and an implementer is free to use, adapt, or ignore any of it as
fits their own application.

- **`file_id` is volume-local, `dir_id` is global, names are not
  identity.** Never use `file_id` alone as a cross-volume key; always pair
  it with `volume_id` (`file_key`). `dir_id` needs no such pairing — it's
  already a complete, stable identity on its own, and never needs
  reconciling across volumes. Never use `file_name` to detect duplicates —
  use `size`+`hash` for file identity.
- **Duplicate detection is a library-wide `size`+`hash` lookup.** The
  format gives the means to detect that content already exists somewhere
  in the library (§3, §9); it intentionally does **not** say whether an
  `F` record implies its bytes are physically written onto that specific
  volume's medium — this standard governs only the index file format (§1),
  not application behavior. Whether to write a duplicate's bytes again
  (redundancy) or skip the physical write (space saving) is a policy
  decision for the application to make — the index format itself doesn't
  answer it. Either way, detecting a match is mechanical; deciding what to
  do about it isn't: surfacing a same-`size`+`hash` match to the archivist
  ("archive it anyway and name it canonical (`AF`), alias it without
  writing the bytes again (`S`, §7 `S`), mark the existing copy gone if
  it's actually bad (`XF`), or keep both as independent copies?") is for
  the application to decide and act on, not something the format decides
  on its own (§7).
- **Counting active files when `AF`/`S` are used.** An `AF` (§7 `AF`)
  names exactly one record canonical among a set of same-`size`+`hash` `F`
  records; for counting purposes, that whole set counts as a single active
  file, not one per copy. An `S` alias (§7 `S`) never counts as a file of
  its own, since it never held content to begin with. Neither mechanism is
  mandatory — two independent `F` records that happen to share content,
  with no `AF` between them, both stay counted as active. Whether and how
  an application materializes an `S` alias as an actual symlink on a
  filesystem is entirely up to that application.
- **`VOLUME_END`'s per-volume fields are not a whole-library checkpoint.**
  `total_vol_files`/`vol_files_size`/`volume_hash` describe only what a
  given volume's own section physically added — they say nothing about
  which of those records are still active once later corrections are
  taken into account (see below). Confirming the library's active file
  count or byte size has no shortcut: resolve the full cumulative log.
  `last_dir_id` and `date_time` on the newest `VOLUME_END`, however, are
  enough on their own to tell which of two `.bdea` files claiming the same
  library is the more current one — e.g. two same-named copies found on
  different volumes. Since `dir_id` only ever grows and is never reused
  (§7 `D`), and `date_time` marks when that section was produced (§6
  `date_time`), whichever file's newest `VOLUME_END` carries the higher
  `last_dir_id` (or later `date_time`) is the more complete, more current
  copy.
- **Quickly verifying an already-written volume.** To confirm a specific
  volume's physical medium still matches its index entry, a reader doesn't
  need to resolve the active view at all: re-read that volume's own `F`
  records, recompute each file's `size`/`hash`, and compare the count and
  summed size against that volume's `total_vol_files`/`vol_files_size`, and
  the concatenated, recomputed hashes (in the same order, via
  `! CONTENT_HASH_ALGORITHM`/`! CONTENT_HASH_ENCODING`) against its
  `volume_hash` (§7 `VOLUME_END`). A match confirms the medium is intact; a
  mismatch pinpoints that something on it has changed since it was written.
  None of this needs a BDEA-aware application: per-file hashes come from
  the OS's own MD5 tool, and `volume_hash` from feeding the concatenated
  hashes to that same tool (§5 "Content hash encoding", §7 `VOLUME_END`) —
  only remember to compare case-insensitively against tools that print
  uppercase.
- **Source names a field can't hold.** Linux and macOS allow almost any
  character in a file or directory name — including `|`, line breaks and
  other control characters, as well as the Windows-reserved characters in
  `! NAME_FORBIDDEN_CHARACTERS`. A producing application scanning such a
  source will meet names the index can't record (§4 "Field content", §5).
  Whether it refuses the file, asks the archivist for a new name, or
  substitutes the offending characters automatically is its own choice;
  what it must not do is write the name as-is. Whatever the policy, a
  name that was changed this way no longer matches the source, so it's
  worth surfacing to the archivist rather than doing silently. The same
  check belongs wherever an application accepts free text into a field —
  `volume_label`, `extra`, or new names given by `RD`/`RF`.
- **Resolving the active view.** A reader maintains working state for the
  library — the directory tree, and each file/alias's current
  `dir_id`/`file_name` — and processes the log as a single left-to-right
  pass with one commit point per volume section:
  - *Within a section*, each record updates that working state
    immediately, in the order it appears (§4): `D`/`F`/`S` add to it;
    `RD`/`RF` overwrite the `parent_dir_id`/`dir_name` (for a directory) or
    `dir_id`/`file_name` (for a file/alias) they target, a later one
    always superseding an earlier one for the same target; `AF` updates
    which record is canonical for its content; `XF`/`XD` remove a record
    or subtree from the view. No lookahead or rescanning is ever needed —
    whichever correction was applied most recently to a given target is,
    by construction, the current one.
  - *On `VOLUME_END`*, this validates against the section's own
    `VOLUME_START` (§7 `VOLUME_START`/`VOLUME_END`). Only once that
    validation succeeds does the section's working state become part of
    the library's durable state; a section that never reaches a valid
    `VOLUME_END` is incomplete and contributes nothing (§7
    `VOLUME_START`/`VOLUME_END`). A later section builds on the durable
    state committed so far, never on another section's still-pending one.

  The currently active files are, at any point in this process, the `F`
  records added so far that haven't since been removed by an `XF`/`XD`,
  adjusted by whichever `AF` most recently named a canonical record for
  their content. This keeps memory bounded to one volume section's worth
  of pending state at a time — never the whole file, and never a second
  pass.
- **Parse the header once per file, before any data.** `! TYPE`/`! RECORD`
  declarations must be read before any data line, since they define the
  field layout for that file (§4). This is a one-time step for a given
  `.bdea` file, not a per-line one: parse the header once, cache the field
  layout it declares, and reuse that cached layout for every data line that
  follows in that same file. Only when a reader switches to a *different*
  `.bdea` file — e.g. another library's index — must it repeat this step,
  since that file's own header, not the previous file's, is the source of
  truth for its field layout. Don't hard-code field order from this
  document alone — treat each file's in-file declarations as the source of
  truth for its `VERSION`.

## 11. Versioning

`! VERSION` on the header identifies the schema version a file conforms to,
written as `MAJOR.MINOR` (e.g. `1.0`, `1.2`, `2.0`) — currently `2.0`.

**History.** `2.0` changed `! CONTENT_HASH_ENCODING` from `base32` (1.0) to
lowercase `hex` (§5), so that recorded hashes match what stock OS tools
print. Every `hash`/`volume_hash` value changes form, which is why this
is a major bump; the record and type layout is otherwise identical to
`1.0`. `2.0` also makes explicit which characters no field may contain
(§4 "Field content"). Files declaring `! VERSION 1.0` remain valid under
[`versions/1.0/BDEA-1.0_STANDARD.md`](../1.0/BDEA-1.0_STANDARD.md). Because the digest itself
is unchanged, a `1.0` file converts to `2.0` mechanically: decode each
`F` record's base32 `hash` and re-encode it as lowercase hex, recompute
every `volume_hash` from the new text (§7 `VOLUME_END`), and update the
two header lines — no archived file needs to be re-read. The one
exception is a field containing a character §4 forbids: a `|` or line
break never parsed correctly under 1.0 either, but another control
character (e.g. a tab in a name) did, and must be removed during
conversion.

**Minor version — backward-compatible, additive.** Bumping only `MINOR`
(e.g. `1.0` → `1.1`) means every file declaring an earlier minor version
within the same `MAJOR` remains fully valid and correctly interpretable
under the new one: nothing already declared changes meaning, shape, or
becomes mandatory. A reader that implements `MAJOR.N` can therefore also
correctly read any file declaring `MAJOR.M` for any `M <= N` — support
accumulates top-down within a major line. Changes that qualify as minor:

- Adding a new optional trailing field to an existing `! RECORD` — the same
  mechanism `extra?` already uses (§7 preamble); a reader unaware of the
  new field simply doesn't see it, exactly as with any other optional
  trailing field.
- Adding a new header declaration (§5) whose absence doesn't change how any
  existing header or record is interpreted — a reader that doesn't
  recognize it can ignore it and still correctly process everything else in
  the file.

**Major version — breaking.** Bumping `MAJOR` (e.g. `1.2` → `2.0`, which
resets `MINOR` to `0`) means files on either side of the boundary aren't
guaranteed compatible: a reader built for `1.x` isn't expected to correctly
read a `2.x` file, or vice versa. Anything that isn't minor-safe by the
criteria above is major, including:

- Changing an existing field's meaning, type, or position within an
  already-declared `! RECORD`.
- Removing a field, a record tag, or a header declaration.
- Turning an existing optional field mandatory, or a mandatory field
  optional.
- Introducing a new record tag or a new mandatory field that a reader
  built for the previous version has no way to skip over and still
  correctly reconstruct the library.

A reader should always trust the specific file's own declared `! VERSION`
(§10 "Parse the header once per file, before any data") over assumptions drawn from this
document alone — this section describes how the two numbers are meant to
behave, not a substitute for reading them.
