# BDEA Index Format Standard

**BDEA — Backup Data Extended Archive.** A long-term, offline archive format for
write-once/high-durability media (optical discs such as BD-R XL, LTO tape, and
similar). This document specifies the `.bdea` index file format. It does not
specify any particular application's behavior (see [§10](#10-notes-for-implementers-bdea-generator--bdea-browser)).

This is the authoritative specification. See
[`samples/Music.bdea`](samples/Music.bdea) for a canonical worked example.

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

**File naming.** A library's index file is named `<LIBRARY>.bdea`, where
`<LIBRARY>` matches the `! LIBRARY` header value (e.g. a library declared as
`! LIBRARY Music` is stored as `Music.bdea`). One file per library; the name
alone identifies which library it belongs to, which is what lets an archive
be built as a plain collection of `<LIBRARY>.bdea` files without any extra
index-of-indexes.

A `.bdea` file is UTF-8 text with two kinds of lines:

- **Declarations** — start with `!`, whitespace-separated tokens:
  `! KEYWORD value...`. These are machine-readable.
- **Comments** — start with `#`, free text, ignored by parsers.
- **Records** — data lines shaped `TAG|field1|field2|...`, pipe-delimited.
  `TAG` is whatever tag a `! RECORD` declaration (§7) defined — one or more
  characters (`VOLUME_START`, `VOLUME_END`, `D`, `F`, `S`, `A`, `X` in the
  current schema).

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
| `! VERSION` | `1` | Format version (see §11). |
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
| `parent_dir_id` | `uint32` | `0` = root of the library's namespace. |
| `file_id` | `uint64` | Local to a volume. |
| `target_file_id` | `uint64` | Same kind as `file_id`; the `F` record an `S` alias reaches for (§7 `S`). |
| `size` | `uint64` | Bytes. |
| `name` | `string max=120` | File or directory name. |
| `hash` | `string max=32` | Encoded content hash (26 chars for base32 MD5; the extra headroom allows for a longer hash if `CONTENT_HASH_ALGORITHM` ever changes). |
| `extra` | `string max=256` | Free-form field on `D`/`F` records (§7 `D`, §7 `F`), e.g. tags. Left empty by any application that has no use for it. |
| `volume_id` | `uint32` | Local to the library. |
| `target_volume_id` | `uint32` | Same kind as `volume_id`; the volume of the `F` record an `S` alias reaches for (§7 `S`). |
| `volume_label` | `string max=120` | Human/physical label printed on the medium. |
| `datetime` | `string max=15` | `YYYY-MM-DD_hhmm` — always when the *producing application* generated the surrounding index record(s), never the physical medium's own write timing (out of scope, §1). Producing one volume's index entries is treated as a single, fast step, so every `datetime` within that volume's section (`VOLUME_START`, `VOLUME_END`, any `A`/`X`) conventionally carries the *same* value — it is not a start/end pair for how long the physical burn took. |
| `count` | `uint64` | Generic non-negative counter kind, shared by `files_added`, `files_changed`, `files_deleted`, `total_files`, and `total_size` below (§7 `VOLUME_END`) — each declared as its own `! TYPE` so a field's type can be looked up by name alone. |
| `files_added` | `uint64` | Same kind as `count`; number of `F` records written in a volume's section (§7 `VOLUME_END`). |
| `files_changed` | `uint64` | Same kind as `count`; number of `A` records written in a volume's section (§7 `VOLUME_END`). |
| `files_deleted` | `uint64` | Same kind as `count`; number of `X` records written in a volume's section (§7 `VOLUME_END`). |
| `total_files` | `uint64` | Same kind as `count`; cumulative count of active files in the library as of a volume (§7 `VOLUME_END`). |
| `total_size` | `uint64` | Same kind as `count`; cumulative byte size of active files in the library as of a volume (§7 `VOLUME_END`). |
| `file_key` | `volume_id file_id` | Composite: the full physical identity of a file record (§3). |

`file_key` is a documentation-level composite — it names the concept of "the
tuple that uniquely locates this record"; the `A`/`X` `! RECORD`
declarations still spell out `volume_id`/`file_id` as separate fields rather
than referencing this composite name directly. Directories have no
equivalent composite: `dir_id` alone is already a directory's full
identity, since it's global across the library.

## 7. Record schema (`! RECORD`)

A field name ending in `?` in a `! RECORD` declaration marks it **optional
and trailing**: a data line using that tag may omit it, having one fewer
pipe-delimited field than declared. As of `! VERSION 1` this applies only to
`extra?` on `D` and `F` (below) — every other field is mandatory. A
conforming parser must accept a `D`/`F` line with or without its trailing
`extra` field, treating a missing one as empty/absent. This is what lets
`extra` be added to a line later (e.g. to attach tags) without invalidating
older lines that predate it, and without requiring every existing line to be
rewritten.

### `VOLUME_START` / `VOLUME_END` — volume boundary

```
! RECORD VOLUME_START volume_id volume_label datetime
! RECORD VOLUME_END volume_id volume_label datetime files_added files_changed files_deleted total_files total_size last_dir_id
```

`VOLUME_START` declares one volume (one physical disc) and opens its
section; `VOLUME_END` closes it. Every `D`, `F`, `S`, `A`, and `X` record
between a `VOLUME_START` and its matching `VOLUME_END` was written as part
of producing that volume:

- `D` records in this range declare any directories newly introduced by
  this volume (§7 `D`) — a directory that already exists in the library
  isn't repeated.
- `F` records in this range derive their `volume_id` from this section (§7
  `F`) — this is the only place a file's volume is recorded.
- `S` records in this range derive their `volume_id` from this section the
  same way `F` does (§7 `S`) — though the *content* they reach for may
  live on any volume, via their own explicit `target_volume_id`.
- `A`/`X` records may appear here even though they can reference a file on
  *any* volume (§7 `A`, `X`) — their own explicit `volume_id` field is what
  matters for them, not their position in the file.

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
  `VOLUME_END`, and any `A`/`X` records the same section contains; it is
  not a start/end pair for the physical burn's duration.

`VOLUME_END`-only fields — this volume's own tally, plus a checkpoint of
the whole library's state as of this volume (§10 explains why these exist
and how an application can use them):

- `files_added` — count of `F` records written in this volume's section.
- `files_changed` — count of `A` records written in this volume's section
  (these may correct files on any volume, not just this one — §7 `A`).
- `files_deleted` — count of `X` records written in this volume's section,
  whether they remove an `F` or an `S` (§7 `X`) — likewise, may reference
  any volume.
- `total_files` — cumulative count of files active in the library as of
  this volume, i.e. the size of the fully reconstructed active view (§10)
  — **not** a raw running total of every `F` record ever written.
  Superseded or deleted records are resolved the same way a full
  reconstruction resolves them, so this can't be maintained by a plain
  per-volume delta (§10). `S` records (§7 `S`) are never counted here — an
  alias adds a new name, not new content; it's already accounted for
  through its target.
- `total_size` — sum of `size` across that same active set; likewise
  unaffected by however many `S` aliases exist.
- `last_dir_id` — highest `dir_id` assigned in the library as of this
  volume; unlike `total_files`/`total_size` this one is a simple monotonic
  maximum, unaffected by `A`/`X` (§7 `D`).

`S` additions themselves aren't tallied by any of the three counts above —
there's no dedicated per-volume counter for aliases as of `! VERSION 1`;
only their removal (via `X`) is folded into `files_deleted`.

```
VOLUME_START|1|Music-001|2026-08-13_1232
...
VOLUME_END|1|Music-001|2026-08-13_1232|9|0|0|9|70477923|4
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
  an `S` uniformly. This is what lets `X` remove an alias the same way it
  removes a file (§7 `X`).
- `dir_id` — where this alias lives in the library's tree (global, exactly
  like `F`'s `dir_id`, including `0` meaning the root, §7 `D`, §7 `F`).
  Unlike `A`/`X`, `S` creates a brand-new tree position, so — unlike `A`/`X`
  — it needs one.
- `name` — this alias's own name at that position.
- `target_volume_id`, `target_file_id` — the existing `F` record whose
  content this alias reaches for (can be on any volume, same pattern as
  `A`/`X`, §3 `file_key`).

`S` carries no `size`/`hash` of its own — the content it reaches for is
already accounted for by its target's `F` record, so an alias adds a new
name to the tree without adding new bytes or new counted content (§7
`VOLUME_END`, §10).

**Resolving a target through `A`/`X`.** An alias's target field identifies
*which content* it reaches for, not a permanently fixed physical record: a
reader resolves it the same way it resolves any other content in the
library, through whatever `A`/`X` history applies to that content (§10
"Read `A`/`X` before trusting `F`"). Concretely: if the target record is
later superseded by an `A` (the archivist renamed/relocated that file —
see below), the alias transparently follows to the new canonical record;
if the target record is later marked `X` and no other copy of that content
survives elsewhere in the library, the alias is dangling, exactly as it
would be if the content were simply gone.

**`S` is not how you rename or relocate a file.** If the same *file* (not
a deliberate second copy) needs a different name or directory going
forward, that's a job for `A`: write a new `F` record at the new
name/location (referencing the same content) and name it canonical with
`A` (§7 `A`, §9). `S` exists only for when the archivist wants the same
content addressable from an additional place *without* writing it again —
never as a substitute for `A`. The format can't tell these two intents
apart mechanically — both start from "the same content, a new name" — so
which one is correct is a matter of what the archivist actually meant, not
something a parser can enforce.

```
S|18|4|cover.jpg|1|1
```

(From `Music-001`'s section in `samples/Music.bdea` — this aliases
`F|1|3|cover.jpg` from a second tree position, `dir_id=4`, the mp3
folder's cover art, instead of archiving those bytes a second time; see
§9.)

### `A` — active

```
! RECORD A volume_id file_id datetime
```

Declares that `(volume_id, file_id)` is the canonical/preferred physical
record to use for its content going forward. Unlike `D`/`F`/`S`, `A` can
point at a file on **any** volume (not just the current one). Typical use: the same
content ends up archived more than once — possibly across several
different volumes (matching `size`+`hash`, e.g. the same file re-added
later under a different name/location, per §9) — `A` names exactly one of
those records as "the" file to present/use going forward. This does **not**
require the other, superseded record(s) to be marked `X` — they can remain
perfectly valid, readable archived copies; they're simply no longer the one
pointed to as canonical. However many copies of that content exist
archive-wide, exactly one stays counted in `total_files`/`total_size`'s
active view once `A` names it (§7 `VOLUME_END`) — the rest drop out of that
count without needing `X`.

- `volume_id`, `file_id` — the record being named canonical (§3 `file_key`).
  Always an `F` record, never an `S` (§7 `S`) — an alias holds no content
  of its own, so it can never be *the* canonical physical copy of
  anything; canonical-for-content only makes sense among records that
  actually hold bytes.
- `datetime` — when this correction was appended to the index (§6
  `datetime`) — not a property of either physical copy; matches the
  enclosing volume's `VOLUME_START`/`VOLUME_END` timestamp.

```
A|2|6|2026-08-14_1111
```

### `X` — deleted

```
! RECORD X volume_id file_id datetime
```

Declares that the record at `(volume_id, file_id)` must be skipped when
reconstructing the library from that point on. Like `A`, it can reference
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
X|1|6|2026-08-14_1111
```

`A` and `X` are independent mechanisms — a given correction may use one,
the other, or both, but one does not imply the other. Together they're how
BDEA supports corrections without rewriting write-once media: an old record
is left physically in place; `A` can redirect which copy is treated as
canonical, and `X` can flag a specific copy as gone.

`A` and `X` are both manual, archivist-issued corrections — nothing in the
format writes them automatically from a `size`+`hash` match. The typical
trigger is the archivist reviewing the same-content matches BDEA-Generator
surfaces (§10) and deciding, case by case: rename/relocate a file going
forward (write `A`), add an alias instead of writing the bytes again if
the same content deliberately belongs in more than one place (write `S`,
§7 `S`), record that a copy has gone bad (write `X`), or simply leave two
independent files that happen to share content alone.

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

VOLUME_END|1|Music-001|2026-08-13_1232|9|0|0|9|70477923|4

# ----------------- Music-002 -----------------

VOLUME_START|2|Music-002|2026-08-14_1111

D|5|1|Sandra Bernardo
...
F|1|6|...

A|2|6|2026-08-14_1111
X|1|6|2026-08-14_1111

VOLUME_END|2|Music-002|2026-08-14_1111|6|1|1|13|83620060|6
```

Each new volume appends: one `VOLUME_START` line, `D` records for any
directories that volume newly introduces (directories that already exist
are not repeated), that volume's `F` records, optionally `S` aliases to
content already archived, optionally `A`/`X` corrections (which may
reference files on *earlier* volumes, not just the new one), and closes
with a matching `VOLUME_END` line.

**Important consequence of write-once media:** because a burned disc can't be
rewritten, its embedded copy of the index is frozen at whatever the library
looked like at burn time. Corrections made later only appear in the index
copies written onto *subsequent* volumes (and in the working master copy on
the computer building the archive) — an older disc's own copy will not show
later `A`/`X` corrections. When reconciling state, prefer the copy from the
highest-numbered (most recent) volume available.

## 9. Worked example

Walking through [`samples/Music.bdea`](samples/Music.bdea):

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
   which is a mistake corrected via `A`. Closing this volume's section,
   `VOLUME_END|1|Music-001|2026-08-13_1232|9|0|0|9|70477923|4` counts `9`
   files, not `10` — the alias doesn't add to the tally (§7 `VOLUME_END`).
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
   - `A|2|6|2026-08-14_1111` — of the two now-known copies of the
     "Classical Violin Concerto" content (`Music-001`'s `F|3|3` and
     `Music-002`'s `F|6|3` from point 2 above, same `size`+`hash`), this
     declares `Music-002`'s copy (`volume=2, file=6`) as the canonical one to
     use going forward. `Music-001`'s copy isn't touched or marked `X` — it
     remains a valid archived copy, just no longer the presented/canonical
     one.
   - `X|1|6|2026-08-14_1111` — a completely separate correction: it declares
     that `Music-001`'s file 6 (`05 Vivaldi Like Violin Solo.flac`) has
     become unreadable/lost on that disc. Since nothing else in the library
     shares its hash, that track is effectively lost from the archive unless
     recovered and re-added later.
4. Closing `Music-002`'s section, `VOLUME_END|2|Music-002|2026-08-14_1111|6|1|1|13|83620060|6`
   records that volume's own tally (`6` files added, `1` change, `1`
   deletion) alongside a checkpoint of the *whole* library. `total_files`
   is `13`, not the `15` `F` records the two volumes have written between
   them (the `S` from point 1 was never one of those `15` to begin with)
   — it already excludes `Music-001`'s `F|3|3` (superseded by the `A`
   above) and `Music-001`'s `F|6|3` (lost per the `X` above):
   `15 - 1 - 1 = 13`. `total_size` (`83620060`) is the sum of `size` across
   that same 13-record active set, and `last_dir_id` (`6`) is the highest
   `dir_id` either volume has assigned so far. A reader that fully
   reconstructs the library from scratch should land on these same three
   numbers — if it doesn't, either the reconstruction or the index has a
   bug (§10).

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
    existing file:
    rename/relocate it (`A`), alias it without writing the bytes again
    (`S`, §7 `S`), or keep both as independent copies?" — not to decide and
    write `A`/`S`/`X` on its own (§7).
- **An `A`-superseded duplicate does not count as its own active file, and
  an `S` alias never counts at all.** Once `A` names one record canonical
  among a set of same-content duplicates — however many volumes hold a
  copy — exactly that one counts toward `total_files`/`total_size`; every
  other record sharing that identity drops out of the active count, though
  it stays physically valid and readable (§7 `A`). §9's worked example
  does this: `Music-001`'s `F|3|3` is excluded once `Music-002`'s `F|6|3`
  is named canonical via `A`. `S` settles the question even more directly
  — `Music-001`'s mp3-folder cover art (§9, point 1) was never a second
  `F` at all, so it was never in the count to begin with. Neither
  mechanism is mandatory, though: the format doesn't require every
  same-`size`+`hash` match to get an `A` or become an `S`. If an archivist
  ever leaves two independent `F` records with matching content
  uncorrected — no `A`, no `S` — both stay counted; whether that's
  deliberate or simply unreviewed is exactly the kind of call `A`/`S`
  exist to let a human make (see above), and the mechanics only care
  whether one was written, not why one hasn't been. Whether *every*
  detected duplicate gets reviewed and resolved one way or the other is a
  BDEA-Generator/archivist workflow question, related to the
  physical-dedup decision above — but once `A` or `S` is written, the
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
- **Read `A`/`X` before trusting `F`.** The set of "currently active" files in
  the library is the `F` records **not** covered by a later `X`, adjusted by
  any `A` that names them explicitly. A correct reader processes the whole
  cumulative log in order, not just the latest volume's own `F` lines.
- **Always parse the header first.** `! TYPE`/`! RECORD` declarations must be
  read before any data line, since they define the field layout for that
  file (§4). Don't hard-code field order from this document alone — treat
  the in-file declarations as the source of truth for a specific file's
  `VERSION`.

## 11. Versioning

`! VERSION` on the header identifies the schema version this file conforms
to (currently `1`, as defined above). Future changes to `! TYPE`/`! RECORD`
shapes should bump this so readers can detect incompatible layouts.
