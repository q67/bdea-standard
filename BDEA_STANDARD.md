# BDEA Index Format Standard

**BDEA — Backup Data Extended Archive.** A long-term, offline archive format for
write-once/high-durability media (optical discs such as BD-R XL, LTO tape, and
similar). This document specifies the `.bdea` index file format. It does not
specify any particular application's behavior (see [§10](#10-notes-for-implementers-bdea-generator--bdea-browser)).

This is the authoritative specification. See
[`samples/music.bdea`](samples/music.bdea) for a canonical worked example.

## 1. Purpose and scope

BDEA's goal is to let a **library** of archived files be reconstructed later —
its full logical structure, not just raw file blobs — without depending on the
original computer, database, or software. The standard covers **only** the
`.bdea` index file format: how a library's structure, volumes, directories,
files, and their history are described in text. It does not cover physical
disc layout, burning software, or backup policy.

## 2. Core concepts

| Term | Meaning |
|---|---|
| **Archive** | The overall BDEA setup: one or more independent libraries. |
| **Library** | A logically isolated collection of data (e.g. `Music`, `Documents`, `Photos`). Has its own directory namespace, its own `.bdea` index, and can be restored independently of any other library. |
| **Volume** | One physical medium (one disc) belonging to a library. A library grows by adding volumes over time (`Music-001`, `Music-002`, ...). |

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

## 4. File structure overview

**File naming.** A library's index file must have the `.bdea` extension;
the rest of the filename is arbitrary — nothing in this standard requires
it to match `! LIBRARY`, its casing, or otherwise encode which library the
file describes. A reader determines that from the `! LIBRARY` header
declaration (§5) inside the file, not from the filename. One file per
library — a file could equally be named `music.bdea`, `music-2026-08-23.bdea`,
or anything else ending in `.bdea`; an application must never infer the
library from the name.

A `.bdea` file is UTF-8 text with two kinds of lines:

- **Declarations** — start with `!`, whitespace-separated tokens:
  `! KEYWORD value...`. These are machine-readable.
- **Comments** — start with `#`, free text, ignored by parsers.
- **Records** — data lines shaped `TAG|field1|field2|...`, pipe-delimited.
  `TAG` is whatever tag a `! RECORD` declaration (§7) defined — one or more
  characters (`VOLUME_START`, `VOLUME_END`, `D`, `F`, `S`, `AF`, `XF`, `XD`,
  `RD`, `RF` in the current schema).

Blank lines are allowed as visual separators (e.g. between volume sections).

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
- A library can, in principle, be browsed in full from a single disc's copy
  of the index, even without the other physical discs present — that copy
  will say which volume physically holds any given file's bytes.

## 5. Header declarations

These appear once, near the top of the file, before any volume section.

| Declaration | Example | Meaning |
|---|---|---|
| `! TEXT_ENCODING` | `utf-8` | Encoding of the `.bdea` file itself. |
| `! TEXT_NORMALIZATION` | `NFC` | Unicode normalization form used for the file's text. |
| `! ARCHIVE_TYPE` | `BDEA` | Format identifier. |
| `! VERSION` | `1.0` | Format version (see §11). |
| `! LIBRARY` | `Music` | Name of this library. |
| `! NAME_ENCODING` | `utf-8` | Encoding used for file/directory `name` fields. |
| `! NAME_NORMALIZATION` | `NFC` | Unicode normalization applied to names. |
| `! NAME_FORBIDDEN_CHARACTERS` | `/ \ : * ? " < > \|` | Characters names must not contain (superset of Windows/exFAT-reserved characters — a safe choice for media meant to be readable from any filesystem). |
| `! CONTENT_HASH_ALGORITHM` | `md5` | Hash algorithm used to identify file content. |
| `! CONTENT_HASH_ENCODING` | `base32` | Encoding used to render the hash as text. |
| `! CONTENT_HASH_SCOPE` | `file-content` | What is hashed (the raw file bytes, as opposed to e.g. name+content). |

This is the canonical, current form of the header, and what new `.bdea`
files should emit.

## 6. Type system (`! TYPE`)

Declares the primitive type of every field used by records, so the file is
self-describing and independently parseable.

| Type name | Kind | Notes |
|---|---|---|
| `dir_id` | `uint32` | Global across the library — assigned once per directory and never reused, regardless of which volume introduced it. Real values start at `1`; `0` is reserved to mean the library's implicit root (§7 `D`, `F`, `S`). |
| `last_dir_id` | `uint32` | Same kind as `dir_id`; highest `dir_id` assigned in the library as of a given volume (§7 `VOLUME_END`). |
| `new_dir_id` | `uint32` | Same kind as `dir_id`; the corrected directory an `RF` correction files a record under (§7 `RF`). |
| `parent_dir_id` | `uint32` | `0` = root of the library's namespace. |
| `new_parent_dir_id` | `uint32` | Same kind as `parent_dir_id`; the corrected parent an `RD` correction gives a directory (§7 `RD`). |
| `file_id` | `uint64` | Local to a volume. |
| `target_file_id` | `uint64` | Same kind as `file_id`; the `F` record an `S` alias reaches for (§7 `S`). |
| `size` | `uint64` | Bytes. |
| `name` | `string max=120` | File or directory name. |
| `new_name` | `string max=120` | Same kind as `name`; the corrected name written by an `RD`/`RF` correction (§7 `RD`, §7 `RF`). |
| `hash` | `string max=32` | Encoded content hash (26 chars for base32 MD5; the extra headroom allows for a longer hash if `CONTENT_HASH_ALGORITHM` ever changes). |
| `extra` | `string max=256` | Free-form field on `D`/`F` records (§7 `D`, §7 `F`), e.g. tags. Left empty by any application that has no use for it. |
| `volume_id` | `uint32` | Local to the library. |
| `target_volume_id` | `uint32` | Same kind as `volume_id`; the volume of the `F` record an `S` alias reaches for (§7 `S`). |
| `volume_label` | `string max=120` | Human/physical label printed on the medium. |
| `datetime` | `string max=15` | `YYYY-MM-DD_hhmm` — always when the *producing application* generated the surrounding index record(s), never the physical medium's own write timing (out of scope, §1). Producing one volume's index entries is treated as a single, fast step, so every `datetime` within that volume's section (`VOLUME_START`, `VOLUME_END`, any `AF`/`XF`/`XD`/`RD`/`RF`) conventionally carries the *same* value — it is not a start/end pair for how long the physical burn took. |
| `count` | `uint64` | Generic non-negative counter kind, shared by `files_added`, `total_files`, and `total_size` below (§7 `VOLUME_END`) — each declared as its own `! TYPE` so a field's type can be looked up by name alone. |
| `files_added` | `uint64` | Same kind as `count`; number of `F` records written in a volume's section (§7 `VOLUME_END`). |
| `total_files` | `uint64` | Same kind as `count`; cumulative count of active files in the library as of a volume (§7 `VOLUME_END`). |
| `total_size` | `uint64` | Same kind as `count`; cumulative byte size of active files in the library as of a volume (§7 `VOLUME_END`). |
| `volume_hash` | `string max=32` | Same shape as `hash` (§6 `hash`); a per-volume checksum derived from other hashes already in the index rather than from a file's own content (§7 `VOLUME_END`). |
| `file_key` | `volume_id file_id` | Composite: the full physical identity of a file record (§3). |

`file_key` is a documentation-level composite — it names the concept of "the
tuple that uniquely locates this record"; the `AF`/`XF`/`RF` `! RECORD`
declarations still spell out `volume_id`/`file_id` as separate fields rather
than referencing this composite name directly. Directories have no
equivalent composite: `dir_id` alone is already a directory's full
identity, since it's global across the library.

## 7. Record schema (`! RECORD`)

A field name ending in `?` in a `! RECORD` declaration marks it **optional
and trailing**: a data line using that tag may omit it, having one fewer
pipe-delimited field than declared. As of `! VERSION 1.0` this applies only to
`extra?` on `D` and `F` (below) — every other field is mandatory. A
conforming parser must accept a `D`/`F` line with or without its trailing
`extra` field, treating a missing one as empty/absent. This is what lets
`extra` be added to a line later (e.g. to attach tags) without invalidating
older lines that predate it, and without requiring every existing line to be
rewritten.

### `VOLUME_START` / `VOLUME_END` — volume boundary

```
! RECORD VOLUME_START volume_id volume_label datetime
! RECORD VOLUME_END volume_id volume_label datetime files_added total_files total_size volume_hash last_dir_id
```

`VOLUME_START` declares one volume (one physical disc) and opens its
section; `VOLUME_END` closes it. Every `D`, `F`, `S`, `AF`, `XF`, `XD`, `RD`,
and `RF` record between a `VOLUME_START` and its matching `VOLUME_END` was
written as part of producing that volume:

- `D` records in this range declare any directories newly introduced by
  this volume (§7 `D`) — a directory that already exists in the library
  isn't repeated.
- `F` records in this range derive their `volume_id` from this section (§7
  `F`) — this is the only place a file's volume is recorded.
- `S` records in this range derive their `volume_id` from this section the
  same way `F` does (§7 `S`) — though the *content* they reach for may
  live on any volume, via their own explicit `target_volume_id`.
- `AF`/`XF`/`RF` records may appear here even though they can reference a
  file on *any* volume (§7 `AF`, `XF`, `RF`) — their own explicit `volume_id`
  field is what matters for them, not their position in the file. `XD`/`RD`
  records likewise may reference a `dir_id` assigned by any earlier volume
  (§7 `XD`, `RD`) — directories need no `volume_id` at all, since `dir_id`
  is already global (§3, §10).

Fields shared by both records:

- `volume_id` — internal numeric id of the volume within the library;
  repeated on `VOLUME_END` so a reader (or a human skimming the file) can
  confirm which section is closing, and so a truncated or corrupted section
  (a `VOLUME_START` with no matching `VOLUME_END`) can be detected.
- `volume_label` — the label written/printed on the physical medium;
  repeated on `VOLUME_END` for the same corruption-detection reason as
  `volume_id` — if the two ends of a section ever disagree, something is
  wrong.
- `datetime` — when the producing application generated this volume's index
  entries (§6 `datetime`). Producing one volume's section is treated as a
  single, fast step — distinct from, and typically much shorter than, the
  physical burn — so this carries the *same* value on `VOLUME_START`,
  `VOLUME_END`, and any `AF`/`XF`/`XD`/`RD`/`RF` records the same section
  contains; it is not a start/end pair for the physical burn's duration.

`VOLUME_END`-only fields — this volume's own tally, plus a checkpoint of
the whole library's state as of this volume (§10 explains why these exist
and how an application can use them):

- `files_added` — count of `F` records written in this volume's section.
- `total_files` — cumulative count of files active in the library as of
  this volume, i.e. the size of the fully reconstructed active view (§10)
  — **not** a raw running total of every `F` record ever written.
  Superseded, deleted, or renamed/relocated records are resolved the same
  way a full reconstruction resolves them, so this can't be maintained by a
  plain per-volume delta (§10). `S` records (§7 `S`) are never counted
  here — an alias adds a new name, not new content; it's already accounted
  for through its target. An `XD` (§7 `XD`) drops its entire subtree from
  this count at once, the same as if every `F`/`S` record under it had its
  own `XF`.
- `total_size` — sum of `size` across that same active set; likewise
  unaffected by however many `S` aliases exist.
- `volume_hash` — a checksum of exactly this volume's own `F` records,
  derived purely from their `hash` values already recorded a few lines up
  in this same section (§6 `hash`) — never by re-reading any file's actual
  content. Computed by concatenating, in the order they appear in this
  volume's section, each `F` record's `hash` field — with no separator
  between them, since every `hash` in a given file is the same fixed
  length (§6 `hash`) for as long as `! CONTENT_HASH_ALGORITHM`/
  `! CONTENT_HASH_ENCODING` stay unchanged, which makes plain
  concatenation unambiguous — then applying `! CONTENT_HASH_ALGORITHM` to
  the resulting bytes and encoding the result with
  `! CONTENT_HASH_ENCODING` (§5) — the same two header declarations
  already used for every per-file hash, just applied over a different
  input. `S` records contribute nothing, since they carry no
  `hash` of their own (§7 `S`); a volume that happens to add no `F` records
  at all uses the hash of an empty input. Unlike
  `total_files`/`total_size`/`last_dir_id`, this is **not** a whole-library
  checkpoint — it's scoped to this one volume, the same way `files_added`
  is, which is what lets it double as a self-contained checksum of what
  that specific physical medium holds.
- `last_dir_id` — highest `dir_id` assigned in the library as of this
  volume; unlike `total_files`/`total_size` this one is a simple monotonic
  maximum, unaffected by `AF`/`XF`/`XD`/`RD`/`RF` (§7 `D`).

`files_added` is the only per-volume tally `! VERSION 1.0` keeps —
`S` additions, and every correction (`AF`, `XF`, `XD`, `RD`, `RF`), have no
dedicated counter of their own. Their effect only shows up indirectly,
through how they move `total_files`/`total_size` between one `VOLUME_END`
checkpoint and the next; a reader wanting to know how many corrections a
given volume's section contains counts the `AF`/`XF`/`XD`/`RD`/`RF` lines in
that section directly, the same way it would count anything else in the
file.

```
VOLUME_START|1|Music-001|2026-08-13_1232
...
VOLUME_END|1|Music-001|2026-08-13_1232|9|9|70477923|6VJWG7TG73DXOYCYXY6R6UXYUU|4
```

### `D` — directory

```
! RECORD D dir_id parent_dir_id name extra?
```

A logical directory in the library's namespace tree.

- `dir_id` — global across the whole library: assigned once, the first time
  a volume introduces this directory, and never reused or re-assigned by a
  later volume — even one that adds more files under it. Assigned values
  start at `1`; the library's root is implicit and never gets a `D` record
  of its own — `0` is reserved to mean "the root" wherever a `dir_id`-shaped
  field references it (`parent_dir_id` below, and `F`/`S`'s own `dir_id`,
  §7 `F`, §7 `S`).
- `parent_dir_id` — the parent directory's `dir_id` (`0` = root). Because
  `dir_id` is global, the parent may well have been declared by an earlier
  volume, not the current one.
- `name` — directory name.
- `extra` — optional, trailing (§7 preamble); free-form (§6 `extra`), e.g.
  tags. Empty or omitted unless an application populates it.

Directories form a single, stable, append-only namespace tree for the whole
library (not one tree per volume). Empty directories may be omitted.
Because `dir_id` is global, a given logical folder (e.g. `Music/Rafael
Krux`) is declared exactly once, by whichever volume first introduces it —
a later volume that adds files under an already-existing folder does not
re-declare it, it simply references the existing `dir_id` from its `F`
records (see the worked example, §9).

```
D|1|0|Music
D|2|1|Rafael Krux
```

### `F` — file

```
! RECORD F file_id dir_id name size hash extra?
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
- `name` — file name (not identity, see §3).
- `size`, `hash` — content identity (§3); used to detect duplicates already
  present anywhere else in the library and to verify integrity.
- `extra` — optional, trailing (§7 preamble); free-form (§6 `extra`), e.g.
  tags. Empty or omitted unless an application populates it.

```
F|1|3|cover.jpg|125142|P625I35ZX7YR6Y4NILBDDE3MQP
F|2|3|...02 Classical Violin Concerto.flac|8541907|VUMGOS3L25V5JEP4LU6OI5PXQG|baroque,epic
```

### `S` — symlink

```
! RECORD S file_id dir_id name target_volume_id target_file_id
```

An alias: a new tree position that reaches for content an `F` record
elsewhere already holds, instead of archiving another physical copy of it.
Typical use: the archivist deliberately wants the same content reachable
from more than one place in the library (e.g. the same `cover.jpg` filed
under two different album folders) without writing those bytes to the
volume a second time.

- `file_id` — this alias's own local id, drawn from the *same* per-volume
  counter as `F`'s `file_id`, not a separate numbering space — so
  `(volume_id, file_id)`, with `volume_id` derived positionally exactly
  like `F`'s (§7 `VOLUME_START`/`VOLUME_END`), addresses either an `F` or
  an `S` uniformly. This is what lets `XF`/`RF` correct an alias the same
  way they correct a file (§7 `XF`, §7 `RF`).
- `dir_id` — where this alias lives in the library's tree (global, exactly
  like `F`'s `dir_id`, including `0` meaning the root, §7 `D`, §7 `F`).
  Unlike `AF`/`XF`, `S` creates a brand-new tree position, so — unlike
  `AF`/`XF` — it needs one.
- `name` — this alias's own name at that position.
- `target_volume_id`, `target_file_id` — the existing `F` record whose
  content this alias reaches for (can be on any volume, same pattern as
  `AF`/`XF`, §3 `file_key`).

`S` carries no `size`/`hash` of its own — the content it reaches for is
already accounted for by its target's `F` record, so an alias adds a new
name to the tree without adding new bytes or new counted content (§7
`VOLUME_END`, §10).

**Resolving a target through `AF`/`XF`/`XD`.** An alias's target field
identifies *which content* it reaches for, not a permanently fixed
physical record: a reader resolves it the same way it resolves any other
content in the library, through whatever `AF`/`XF`/`XD` history applies to
that content (§10 "Read `AF`/`XF`/`XD`/`RD`/`RF` before trusting `F`/`D`").
Concretely: if the target record is later superseded by an `AF` (a genuine
second physical copy of that content is named canonical instead — see
below), the alias transparently follows to the new canonical record; if
the target record is later marked `XF` (or its containing directory is
removed via `XD`) and no other copy of that content survives elsewhere in
the library, the alias is dangling, exactly as it would be if the content
were simply gone. `RF`/`RD` corrections to the target's own name/location
don't affect the alias at all — it addresses the target by `file_key`, not
by name, so the target can be renamed or relocated freely without touching
anything that aliases it.

**`S` is not how you rename or relocate a file.** If the same *file* (not
a deliberate second copy) needs a different name or directory going
forward, that's a job for `RF`: correct the existing record's `dir_id`/
`name` in place, without writing its content again (§7 `RF`). `S` exists
only for when the archivist wants the same content addressable from an
additional place *without* writing it again — never as a substitute for
`RF`. The format can't tell these two intents apart mechanically — both
start from "the same content, a new name" — so which one is correct is a
matter of what the archivist actually meant, not something a parser can
enforce.

```
S|18|4|cover.jpg|1|1
```

(From `Music-001`'s section in `samples/music.bdea` — this aliases
`F|1|3|cover.jpg` from a second tree position, `dir_id=4`, the mp3
folder's cover art, instead of archiving those bytes a second time; see
§9.)

### `AF` — active

```
! RECORD AF volume_id file_id datetime
```

Declares that `(volume_id, file_id)` is the canonical/preferred physical
record to use for its content going forward. Unlike `D`/`F`/`S`, `AF` can
point at a file on **any** volume (not just the current one). Typical use: the same
content ends up archived more than once — possibly across several
different volumes (matching `size`+`hash`, e.g. the same file re-added
later under a different name/location, per §9) — `AF` names exactly one of
those records as "the" file to present/use going forward. This does **not**
require the other, superseded record(s) to be marked `XF` — they can remain
perfectly valid, readable archived copies; they're simply no longer the one
pointed to as canonical. However many copies of that content exist
archive-wide, exactly one stays counted in `total_files`/`total_size`'s
active view once `AF` names it (§7 `VOLUME_END`) — the rest drop out of that
count without needing `XF`.

- `volume_id`, `file_id` — the record being named canonical (§3 `file_key`).
  Always an `F` record, never an `S` (§7 `S`) — an alias holds no content
  of its own, so it can never be *the* canonical physical copy of
  anything; canonical-for-content only makes sense among records that
  actually hold bytes.
- `datetime` — when this correction was appended to the index (§6
  `datetime`) — not a property of either physical copy; matches the
  enclosing volume's `VOLUME_START`/`VOLUME_END` timestamp.

```
AF|2|6|2026-08-14_1111
```

### `XF` — deleted (file/alias)

```
! RECORD XF volume_id file_id datetime
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
- `datetime` — when this correction was appended to the index (§6
  `datetime`) — not when the medium actually failed or was noticed as bad;
  matches the enclosing volume's `VOLUME_START`/`VOLUME_END` timestamp.

```
XF|1|6|2026-08-14_1111
```

### `XD` — deleted (directory)

```
! RECORD XD dir_id datetime
```

The directory-scoped counterpart to `XF`: declares that `dir_id`, and
everything beneath it in the library's tree — every descendant directory
and every `F`/`S` record filed anywhere in that subtree, regardless of
which volume(s) they were written on — must be skipped when reconstructing
the library from that point on. Written when a whole branch is gone at
once (e.g. the surviving, still-readable part of a damaged volume didn't
include this branch), instead of writing one `XF` per file it contained.

- `dir_id` — the directory being marked gone, and the root of the subtree
  it takes with it (§7 `D`). Because `dir_id` is never reused (§7 `D`),
  this alone is enough to identify the target — no companion field is
  needed the way `XF` needs `volume_id` alongside `file_id`.
- `datetime` — when this correction was appended to the index (§6
  `datetime`); matches the enclosing volume's `VOLUME_START`/`VOLUME_END`
  timestamp, exactly like `XF`.

`XD` marks everything *currently* filed under `dir_id` as gone as of this
correction — it is not a claim that `dir_id` itself is retired. A later
volume can still file new `F`/`D` records under the same `dir_id`, exactly
as it could before the `XD`; only what was already there at the time of
the correction is affected.

Like `XF`, this is a statement about the index's active view, not a claim
about the medium's material condition — the format doesn't distinguish a
truly unrecoverable copy from one an archivist simply chose to treat as
gone.

```
XD|6|2026-08-20_0900
```

(From `Music-003`'s section in `samples/music.bdea` — the surviving,
readable part of `Music-002`'s disc didn't include Sandra Bernardo's
"Tropico Ideal (2018)" album, so `dir_id=6` and its five `F` records are
dropped from the active view at once; see §9.)

### `RD` — renamed/relocated (directory)

```
! RECORD RD dir_id new_parent_dir_id new_name datetime
```

Corrects an existing directory's `parent_dir_id` and/or `name` (§7 `D`)
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
- `new_name` — the corrected name (§6 `new_name`); repeat the existing
  `name` unchanged if this correction is a pure move.
- `datetime` — when this correction was appended to the index (§6
  `datetime`); matches the enclosing volume's `VOLUME_START`/`VOLUME_END`
  timestamp, exactly like `XF`.

If more than one `RD` targets the same `dir_id` over time, the one
appearing latest in the cumulative log wins — a reader always resolves a
directory's current `parent_dir_id`/`name` from the most recent `RD` that
names it, or from its original `D` record if none exists (§10).

```
RD|3|9|(flac)|2026-08-20_0900
RD|4|9|(mp3)|2026-08-20_0900
```

(From `Music-003`'s section in `samples/music.bdea` — a new grouping
directory `D|9|2|Rafael Krux - The Epic Baroque Album` is introduced under
`Rafael Krux` (`dir_id=2`), and the two format-folders that used to sit
directly under it, `dir_id=3` and `dir_id=4`, are reparented under `9` and
shortened to `(flac)`/`(mp3)` — both fields change in the same correction.
Every `F`/`S` record already filed under `3`/`4` (§9, points 1–2) moves
with them without a single one of those records being touched; see §9.)

### `RF` — renamed/relocated (file/alias)

```
! RECORD RF volume_id file_id new_dir_id new_name datetime
```

Corrects an existing `F` or `S` record's `dir_id` and/or `name` going
forward, without writing its content again. This is the mechanism for a
pure naming or filing mistake: the record's `size`+`hash` (for an `F`) or
target (for an `S`) never change, only where it sits in the tree and what
it's called there.

- `volume_id`, `file_id` — the record being corrected (§3 `file_key`);
  addresses either an `F` or an `S`, the same way `AF`/`XF` do (§7 `S`).
- `new_dir_id` — the corrected directory (§6 `new_dir_id`); repeat the
  existing `dir_id` unchanged if this correction is a pure rename.
- `new_name` — the corrected name (§6 `new_name`); repeat the existing
  `name` unchanged if this correction is a pure move.
- `datetime` — when this correction was appended to the index (§6
  `datetime`); matches the enclosing volume's `VOLUME_START`/`VOLUME_END`
  timestamp, exactly like `XF`.

If more than one `RF` targets the same `(volume_id, file_id)` over time,
the most recent one wins, the same way `RD` resolves for a `dir_id` (§10).
`RF` is unrelated to `AF`: `AF` picks which of several independent physical
copies of the *same content* is canonical; `RF` corrects where *one
specific* record sits in the tree, whether or not any duplicate of it
exists.

```
RF|1|18|4|folder.jpg|2026-08-20_0900
```

(From `Music-003`'s section in `samples/music.bdea` — renames the mp3
folder's cover-art alias, `S|18|4|cover.jpg|1|1` from `Music-001` (§7 `S`),
to `folder.jpg`; `dir_id` stays `4`, only `name` changes. Demonstrates `RF`
correcting an `S` record, not just an `F`; see §9.)

`AF`, `XF`, `XD`, `RD`, and `RF` are independent mechanisms — a given
correction may use any combination of them, but none implies another.
Together they're how BDEA supports corrections without rewriting
write-once media: an old record is left physically in place; `AF` redirects
which copy is treated as canonical, `XF`/`XD` flag a copy (or a whole
branch) as gone, and `RD`/`RF` fix a naming or filing mistake in place.

All five are manual, archivist-issued corrections — nothing in the format
writes them automatically. `AF`/`S`/`XF` are the responses to a genuine
same-`size`+`hash` match that BDEA-Generator surfaces (§10): name one copy
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
# ----------------- Music-001 -----------------

VOLUME_START|1|Music-001|2026-08-13_1232

D|1|0|Music
...
F|1|3|cover.jpg|125142|...
...
S|18|4|cover.jpg|1|1

VOLUME_END|1|Music-001|2026-08-13_1232|9|9|70477923|6VJWG7TG73DXOYCYXY6R6UXYUU|4

# ----------------- Music-002 -----------------

VOLUME_START|2|Music-002|2026-08-14_1111

D|5|1|Sandra Bernardo
...
F|1|6|...

AF|2|6|2026-08-14_1111
XF|1|6|2026-08-14_1111

VOLUME_END|2|Music-002|2026-08-14_1111|6|13|83620060|NMKRK63FVB7PPOLNTD7S5BL27A|6
```

Each new volume appends: one `VOLUME_START` line, `D` records for any
directories that volume newly introduces (directories that already exist
are not repeated), that volume's `F` records, optionally `S` aliases to
content already archived, optionally `AF`/`XF`/`XD`/`RD`/`RF` corrections
(which may reference files or directories from *earlier* volumes, not just
the new one), and closes with a matching `VOLUME_END` line.

**Important consequence of write-once media:** because a burned disc can't be
rewritten, its embedded copy of the index is frozen at whatever the library
looked like at burn time. Corrections made later only appear in the index
copies written onto *subsequent* volumes (and in the working master copy on
the computer building the archive) — an older disc's own copy will not show
later `AF`/`XF`/`XD`/`RD`/`RF` corrections. When reconciling state, prefer
the copy from the highest-numbered (most recent) volume available.

## 9. Worked example

Walking through [`samples/music.bdea`](samples/music.bdea):

1. **Music-001** is burned first. It declares directories for
   `Music/Rafael Krux/...(flac)` and `.../...(mp3)`, and files for a cover
   image plus several flac/mp3 tracks. Each flac track's `F` record carries
   `baroque,epic` in its trailing `extra` field (§6, §7 `F`) — a
   concrete illustration of `extra` being free-form; nothing else in the
   sample interprets these tags, they're just carried along. The mp3
   folder's cover art isn't a
   second physical copy: `S|18|4|cover.jpg|1|1` aliases the flac folder's
   `F|1|3|cover.jpg` instead of writing those bytes again (§7 `S`) — the
   deliberate-duplication case `S` exists for, as opposed to point 2 below,
   which is a mistake corrected via `AF`. Closing this volume's section,
   `VOLUME_END|1|Music-001|2026-08-13_1232|9|9|70477923|6VJWG7TG73DXOYCYXY6R6UXYUU|4`
   counts `9` files, not `10` — the alias doesn't add to the tally (§7
   `VOLUME_END`). Its second-to-last field, `6VJWG7TG73DXOYCYXY6R6UXYUU`, is
   `volume_hash`: concatenate those same 9 `F` records' own `hash` values,
   in the order they appear (no separator needed, since they're all the
   same fixed length), then hash and encode that text the same way
   `CONTENT_HASH_ALGORITHM`/`CONTENT_HASH_ENCODING` (§5) already hash file
   content. No archived byte is touched to produce it — it's built purely
   from hashes already sitting in the index.
2. **Music-002** is burned later. It declares the one genuinely new folder
   tree it needs (`Music/Sandra Bernardo/...`, `dir_id` 5 and 6) and adds
   that artist's files there. It also adds one more file directly under
   `Music/Rafael Krux/Rafael Krux - The Epic Baroque Album (flac)` — the
   directory `Music-001` already declared as `dir_id=3` — without
   re-declaring it:
   (`F|6|3|...02 Classical Violin Concerto number 3.flac|8541907|VUMGOS3L25V5JEP4LU6OI5PXQG|baroque,epic`)
   — whose `size`+`hash` are **identical** to `Music-001`'s
   `F|3|3|...02 Classical Violin Concerto.flac`. This is the concrete case
   the format's content-identity model (§3) exists to catch: same content,
   different volume, different (mistaken) name — still recognizable as one
   piece of content via `size`+`hash`. What an application chooses to *do*
   about that (skip re-writing the bytes, write them anyway for redundancy,
   warn the user, etc.) is implementation policy — see §10.
3. At the end of the `Music-002` section, two *unrelated* corrections are
   recorded together:
   - `AF|2|6|2026-08-14_1111` — of the two now-known copies of the
     "Classical Violin Concerto" content (`Music-001`'s `F|3|3` and
     `Music-002`'s `F|6|3` from point 2 above, same `size`+`hash`), this
     declares `Music-002`'s copy (`volume=2, file=6`) as the canonical one to
     use going forward. `Music-001`'s copy isn't touched or marked `XF` — it
     remains a valid archived copy, just no longer the presented/canonical
     one.
   - `XF|1|6|2026-08-14_1111` — a completely separate correction: it declares
     that `Music-001`'s file 6 (`05 Vivaldi Like Violin Solo.flac`) has
     become unreadable/lost on that disc. Since nothing else in the library
     shares its hash, that track is effectively lost from the archive unless
     recovered and re-added later.
4. Closing `Music-002`'s section, `VOLUME_END|2|Music-002|2026-08-14_1111|6|13|83620060|NMKRK63FVB7PPOLNTD7S5BL27A|6`
   records that volume's own tally (`6` files added) alongside a checkpoint
   of the *whole* library. `total_files`
   is `13`, not the `15` `F` records the two volumes have written between
   them (the `S` from point 1 was never one of those `15` to begin with)
   — it already excludes `Music-001`'s `F|3|3` (superseded by the `AF`
   above) and `Music-001`'s `F|6|3` (lost per the `XF` above):
   `15 - 1 - 1 = 13`. `total_size` (`83620060`) is the sum of `size` across
   that same 13-record active set, and `last_dir_id` (`6`, the final field)
   is the highest `dir_id` either volume has assigned so far. A reader that
   fully reconstructs the library from scratch should land on these same
   three numbers — if it doesn't, either the reconstruction or the index
   has a bug (§10). The second-to-last field, `NMKRK63FVB7PPOLNTD7S5BL27A`,
   is this section's own `volume_hash`, built the same way as
   `Music-001`'s but from only this volume's `6` `F` hashes — it does not
   extend or combine with `Music-001`'s `volume_hash`, since (unlike
   `total_files`/`total_size`/`last_dir_id`) `volume_hash` is scoped per
   volume, not a whole-library checkpoint.
5. **Music-003** is burned later still, and adds no new archived content at
   all — its section is nothing but corrections, which is a perfectly valid
   volume:
   - `D|9|2|Rafael Krux - The Epic Baroque Album` introduces a new grouping
     directory under `Rafael Krux` (`dir_id=2`), then
     `RD|3|9|(flac)|2026-08-20_0900` and `RD|4|9|(mp3)|2026-08-20_0900`
     (§7 `RD`) reparent the two existing format-folders under it and shorten
     their names — the archivist decided the album name shouldn't be
     repeated at both levels. Every `F`/`S` record `Music-001`/`Music-002`
     already filed under `dir_id=3`/`4` relocates with them; none of those
     records is touched.
   - `RF|1|18|4|folder.jpg|2026-08-20_0900` (§7 `RF`) renames
     `Music-001`'s `S|18|4|cover.jpg|1|1` alias to `folder.jpg`, without
     touching its target or its `dir_id` — a pure naming fix, and a
     concrete case of `RF` correcting an `S` rather than an `F`.
   - `XD|6|2026-08-20_0900` (§7 `XD`) declares that Sandra Bernardo's
     "Tropico Ideal (2018)" directory (`dir_id=6`, from `Music-002`) — and
     the five `F` records filed under it — didn't survive whatever made
     the archivist revisit that disc. No `XF` is needed per file.
   - Closing this volume,
     `VOLUME_END|3|Music-003|2026-08-20_0900|0|8|53853523|2QOYZWMPACZAJ2MABGMOZ6CCPY|9`
     records `0` for `files_added` — no `F` record was written — while
     `total_files` drops from `13` to `8` (the five files under the `XD`'d
     `dir_id=6`) and `total_size` drops from `83620060` to `53853523` by
     the same five records' combined `size`. `last_dir_id` becomes `9`,
     the one new `D` this volume introduced. `volume_hash`
     (`2QOYZWMPACZAJ2MABGMOZ6CCPY`) is the hash of an *empty* input (§7
     `VOLUME_END`), since this volume's section contains no `F` record to
     concatenate a hash from.

## 10. Notes for implementers (BDEA-Generator / BDEA-Browser)

These are consequences of the format worth keeping in mind when building
applications against it — not additional format rules:

- **`file_id` is volume-local, `dir_id` is global, names are not
  identity.** Never use `file_id` alone as a cross-volume key; always pair
  it with `volume_id` (`file_key`). `dir_id` needs no such pairing — it's
  already a complete, stable identity on its own, and never needs
  reconciling across volumes. Never use `name` to detect duplicates — use
  `size`+`hash` for file identity.
- **Duplicate detection is a library-wide `size`+`hash` lookup.** The format
  gives you the means to detect that content already exists somewhere in the
  library (§3, §9). It intentionally does **not** say whether a `F` record
  implies the bytes are physically burned onto that specific volume's disc —
  this standard governs only the index file format (§1), not application
  behavior. Whether BDEA-Generator writes the bytes again (redundancy) or
  skips the physical copy when a duplicate is found (space saving) is a
  policy decision to make explicitly when BDEA-Generator is designed — it
  is not answered by the index format itself.
  - If the physical-copy policy is "skip when duplicate,"
    BDEA-Generator/BDEA-Browser will need a convention for *which* volume
    is guaranteed to hold the physical bytes for a given hash (e.g. "the
    earliest volume where that hash first appears") — that convention
    doesn't exist yet and should be decided before BDEA-Generator relies
    on it.
  - Either way, detecting a match is mechanical; deciding what to do about
    it isn't. BDEA-Generator's job is to surface a same-`size`+`hash`
    match to the archivist — e.g. "this looks like a duplicate of an
    existing file: archive it anyway and name it canonical (`AF`), alias it
    without writing the bytes again (`S`, §7 `S`), mark the existing copy
    gone if it's actually bad (`XF`), or keep both as independent copies?"
    — not to decide and write `AF`/`S`/`XF` on its own (§7). A pure naming
    or filing mistake — nothing duplicated — isn't this flow at all; that's
    `RD`/`RF` (§7 `RD`, §7 `RF`), issued whenever the archivist notices the
    mistake, independent of duplicate detection.
- **An `AF`-superseded duplicate does not count as its own active file, and
  an `S` alias never counts at all.** Once `AF` names one record canonical
  among a set of same-content duplicates — however many volumes hold a
  copy — exactly that one counts toward `total_files`/`total_size`; every
  other record sharing that identity drops out of the active count, though
  it stays physically valid and readable (§7 `AF`). §9's worked example
  does this: `Music-001`'s `F|3|3` is excluded once `Music-002`'s `F|6|3`
  is named canonical via `AF`. `S` settles the question even more directly
  — `Music-001`'s mp3-folder cover art (§9, point 1) was never a second
  `F` at all, so it was never in the count to begin with. Neither
  mechanism is mandatory, though: the format doesn't require every
  same-`size`+`hash` match to get an `AF` or become an `S`. If an archivist
  ever leaves two independent `F` records with matching content
  uncorrected — no `AF`, no `S` — both stay counted; whether that's
  deliberate or simply unreviewed is exactly the kind of call `AF`/`S`
  exist to let a human make (see above), and the mechanics only care
  whether one was written, not why one hasn't been. Whether *every*
  detected duplicate gets reviewed and resolved one way or the other is a
  BDEA-Generator/archivist workflow question, related to the
  physical-dedup decision above — but once `AF` or `S` is written, the
  counting rule itself is fixed, not a choice.
- **`VOLUME_END`'s summary fields double as a rebuild checkpoint.** An
  application that keeps a persistent database mirroring the library,
  instead of re-parsing the whole `.bdea` file on every run, can normally
  just append the newest volume's section to that database rather than
  reprocessing history from scratch. Before doing that append, compare the
  database's own current `(total_files, total_size, last_dir_id)` against
  the values on the *latest* `VOLUME_END` already applied. A match means
  it's safe to append just the new section; a mismatch means the database
  has drifted (edited outside the app, corrupted, or simply a different
  copy than expected) and needs a full resync instead — rebuild the
  database from the index, rebuild the index from the database, or track
  down whichever `.bdea` copy is actually the newest (§8).
- **Read `AF`/`XF`/`XD`/`RD`/`RF` before trusting `F`/`D`.** The set of
  "currently active" files in the library is the `F` records **not**
  covered by a later `XF`, and not filed (directly or transitively) under a
  directory covered by a later `XD`, adjusted by any `AF` that names them
  explicitly. A file's or alias's presented `dir_id`/`name` is whatever the
  most recent `RF` for that `(volume_id, file_id)` says, or its own `F`/`S`
  record if no `RF` exists; a directory's presented `parent_dir_id`/`name`
  resolves the same way through `RD`. A correct reader processes the whole
  cumulative log in order, not just the latest volume's own `F`/`D` lines.
- **Always parse the header first.** `! TYPE`/`! RECORD` declarations must be
  read before any data line, since they define the field layout for that
  file (§4). Don't hard-code field order from this document alone — treat
  the in-file declarations as the source of truth for a specific file's
  `VERSION`.

## 11. Versioning

`! VERSION` on the header identifies the schema version a file conforms to,
written as `MAJOR.MINOR` (e.g. `1.0`, `1.2`, `2.0`) — currently `1.0`.

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
(§10 "Always parse the header first") over assumptions drawn from this
document alone — this section describes how the two numbers are meant to
behave, not a substitute for reading them.
