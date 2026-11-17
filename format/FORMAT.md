# Writers Rolltop file format 1

This document describes the files Writers Rolltop writes and reads: the book archive (`.rolltop`) and the library backup (`.rolltoplibrary`). It describes format version 1 as the Apple code in milestone 1 writes it. It is started in milestone 1 and grows with each record type. The behaviour behind it is defined by the product spec (sections 4 and 7); where this document and the spec differ, the spec is right and this document is wrong.

Items marked **Open point** are things the code or the spec does not settle yet. They are not decisions.

Not yet in this document: the JSON Schema, the fixture book, the bible record types, and the single-book import rules (Replace, Keep both).

## 1. Conventions

- **Ids** are UUIDs written lower case (`8-4-4-4-12` hex digits). They are random, never reused and never changed. One exception: a book's "Recovered" chapter has the id UUID version 5 of the text `<book id>:recovered` (book id lower case) in the URL namespace.
- **Times** are UTC, ISO 8601 with milliseconds and a `Z`: `YYYY-MM-DDTHH:MM:SS.mmmZ`, for example `2026-11-16T09:30:05.123Z`. Times are for display only; no rule compares clocks.
- **Numbers** that are counts are whole JSON numbers below 10^15.
- **JSON files** are UTF-8. The Apple code writes them in a canonical form: object keys sorted by their UTF-8 bytes, no white space, strings escaped only for `"`, `\` and code points below U+0020 (`\b \t \n \f \r`, others as `\u00xx` lower case), whole numbers written as integers. Readers must accept any valid JSON.
- **Unknown fields**: a reader keeps every field it does not know, on every record, in a scene's document, in each block and in each run, and writes it back unchanged. See section 6.
- **Optional fields** are left out when they have no value. A reader treats a field that is `null` as absent.

## 2. Archive layout (`.rolltop`, one book)

A ZIP file. Entry names use `/`. Every file below is always written; a kind with no records is an empty list `[]`.

```
manifest.json
book.json
manuscript/nodes.json
manuscript/scenes/<scene id>.json
bible/entries.json
bible/templates.json
bible/relationships.json
bible/events.json
bible/changes.json
bible/facts.json
bible/questions.json
notes.json
attachments.json
attachments/<sha256>.<extension>
```

- `manuscript/scenes/<scene id>.json` is written by the Apple code only for a scene that has saved text. `<scene id>` is the node id, lower case. **Open point:** spec 4.2 says scene text is made with its scene, which would mean one file per scene; the Apple code and spec differ here until the owner rules.
- Items in the book's Trash are included, marked with `deletedAt`, so a live record that points at a trashed one still resolves and Restore still works after an import.
- Not included: scene history, conflict copies, sync state, device settings.
- The Apple code stores entries without compression. Readers must accept compressed entries.
- In milestone 1 the `bible/*.json` files and `notes.json` are always `[]`, because those record types do not exist yet. An app refuses a file that holds anything it cannot keep (spec 7.5): the Apple reader refuses a file where one of those lists holds records, a file that holds a path outside this layout (even when the manifest lists it), and an attachment file that no attachment record uses. Nothing is dropped.

### 2.1 `manifest.json`

An object:

| Field | Type | Meaning |
|---|---|---|
| `formatVersion` | number | `1` |
| `app` | string | `"Writers Rolltop"` |
| `appVersion` | string | version of the exporting app |
| `exportedAt` | string | time of export |
| `bookId` | string | id of the book, lower case |
| `files` | object | every other file in the archive: its path mapped to the lower-case hex SHA-256 (64 characters) of its bytes |

A file made by a newer format version than the reader understands is refused with "Update Writers Rolltop to open this file". Older supported versions are upgraded on import. (Only version 1 exists.)

### 2.2 `book.json`

An object with a `book` part and, usually, a `shelf` part:

- `book`: the complete book record (section 4.3).
- `shelf`: `id` (string), `kind` (`series` or `standAlone`), `title` (string), `orderKey` (string): the shelf the book was on and its position among shelves. A writer leaves the `shelf` part out for a book in the Trash whose shelf is gone; a reader accepts either (section 7).

`carriedFrom` is a field of the book record. The book record has no such field in milestone 1; if a file carries one, it is an unknown field and is kept (section 6).

**Open point:** unknown fields on the `book.json` object itself and on its `shelf` part are not kept by the Apple reader (only the book record's own are).

### 2.3 Record lists

`manuscript/nodes.json` is a list of node records. `attachments.json` is a list of attachment records, including those in the Trash. The Apple writer sorts each list by record id; a reader must not depend on the order. Order among siblings comes from `orderKey`.

### 2.4 Attachment files

`attachments/<sha256>.<extension>`. `<sha256>` is the attachment record's `sha256`: exactly 64 lower-case hexadecimal characters, the SHA-256 of the file's bytes. `<extension>` is the extension of the record's `fileName`, lower case, ASCII letters and digits only; when the file name has none that qualifies, there is no extension and no dot. Each record must have its file. Two records with the same bytes and extension share one file. The largest single file is 50 MB, meaning 50,000,000 bytes.

## 3. Backup layout (`.rolltoplibrary`, a whole library)

A ZIP file holding:

```
library.json
books/<book id>.rolltop
```

`library.json` is an object: `library` (the library record) and `shelves` (a list of the shelf records, including shelves in the Trash). There is one `books/<book id>.rolltop` for every book, including books in the Trash; each is a complete archive as in section 2, stored whole inside the backup. A book's shelf (`shelfId`) must be one of the shelves in `library.json`. The backup has no manifest of its own: each book archive has one.

## 4. Records

Field names are the JSON names. Strings are UTF-8. "Optional" means the field may be left out.

### 4.1 Library record

| Field | Type | Notes |
|---|---|---|
| `id` | id | |
| `name` | string | |
| `formatVersion` | number | |
| `generation` | id | optional |
| `createdAt` | time | |
| `updatedAt` | time | |

### 4.2 Shelf

| Field | Type | Notes |
|---|---|---|
| `id` | id | |
| `kind` | string | `series` or `standAlone` |
| `title` | string | |
| `orderKey` | string | position among shelves |
| `createdAt`, `updatedAt` | time | |
| `deletedAt` | time | optional; set when in the Trash |

### 4.3 Book

| Field | Type | Notes |
|---|---|---|
| `id` | id | |
| `shelfId` | id | the shelf the book is on |
| `title` | string | |
| `orderKey` | string | position on its shelf |
| `authorName` | string | |
| `description` | string | |
| `createdAt`, `updatedAt` | time | |
| `deletedAt` | time | optional |

### 4.4 Node (part, chapter or scene)

| Field | Type | Notes |
|---|---|---|
| `id` | id | |
| `bookId` | id | |
| `parentId` | id | optional; absent at the top level |
| `kind` | string | `part`, `chapter` or `scene` |
| `title` | string | |
| `orderKey` | string | position among siblings |
| `status` | string | `toDo`, `draft`, `revised` or `done` |
| `synopsis` | string | |
| `createdAt`, `updatedAt` | time | |
| `deletedAt` | time | optional |

A part sits at the top level and holds chapters; a chapter sits at the top level or in a part and holds scenes; a scene is always in a chapter. Every `parentId` must be the id of a node in the same file (a node in the Trash counts).

### 4.5 Scene text

| Field | Type | Notes |
|---|---|---|
| `nodeId` | id | the scene node; equals `<scene id>` in the file name |
| `bookId` | id | |
| `document` | object | section 5 |
| `wordCount` | number | section 5.2 |
| `updatedAt` | time | scene text carries `updatedAt` only |

### 4.6 Attachment

| Field | Type | Notes |
|---|---|---|
| `id` | id | |
| `bookId` | id | |
| `sha256` | string | 64 lower-case hex characters |
| `fileName` | string | |
| `mediaType` | string | |
| `byteCount` | number | size of the file |
| `createdAt` | time | attachments carry `createdAt` only |
| `deletedAt` | time | optional |

### 4.7 Conflict copy

Not written to archives or backups. Its fields in the Apple code, for sync: `id`, `sceneId`, `bookId`, `document`, `deviceName`, `createdAt` (conflict copies carry `createdAt` only).

### 4.8 Fields of the spec not yet present

**Open point:** spec 4.3 lists fields that the milestone 1 records do not have yet. Book: cover image, overview (logline, premise, themes, arcs, principles), target word count, daily word goal, `carriedFrom`. Node: point-of-view entry, setting entry, planned words, planned beats. They arrive with the milestones that build them; until then a file carrying them keeps them as unknown fields.

### 4.9 Order keys

`orderKey` is a text key that sorts siblings in plain byte order; equal keys sort by id. The Apple code writes keys of the digits `0-9A-Za-z` (ASCII order) that do not end in `0`. **Open point:** whether other editions must use the same key scheme, or only keep keys they read in order.

## 5. Scene text

### 5.1 Document

`document` is an object with `blocks`, a list. Unknown fields on the document are kept.

A block is an object: `id` (id), `kind`, `runs` (a list). Block kinds: `paragraph`, `heading` (one level), `bulletItem`, `numberedItem`, `sceneBreak`. Lists are one level deep; list items do not nest. A scene break has no text. Unknown fields on a block are kept.

A run is an object: `text` (string), and optional `bold`, `italic`, `underline` (true when set; the Apple code leaves them out when false and reads a missing one as false), and optional `entryId` (id), a link to a bible entry. Marks may be combined. Unknown fields on a run are kept. A line break inside a paragraph is a `\n` in `text`.

Nothing else is stored: no fonts, sizes, colours or alignment.

The Apple editor saves a document in a normalized form (adjacent runs with the same marks, `entryId` and unknown fields are merged; a run with empty text and no unknown fields is dropped). A reader keeps a document exactly as given and does not normalize it.

### 5.2 Word count

`wordCount` follows spec 4.4. Split the scene's text on Unicode white space and count the pieces that contain at least one letter or digit.

- White space is any character with the Unicode `White_Space` property, including the no-break space (U+00A0) and the ideographic space (U+3000). The zero-width space (U+200B) is not white space and does not split a word.
- A letter is any character in Unicode general category L; a digit is any character in category Nd. Other numeric characters (such as ½, ² and Ⅳ) are not digits.
- Text inside a scene break block is not counted.

The scene's text is its blocks' runs joined, with blocks separated by a line break (which is white space).

## 6. Unknown fields

A reader keeps every field it does not know and writes it back unchanged: on every record in the lists and files above, on a scene's `document`, on each block and on each run. The Apple code keeps them in the stored JSON of each record, so they survive the database, a backup and sync.

Not covered by this rule in the Apple reader: the `manifest.json` object, the `book.json` wrapper (section 2.2) and the `library.json` wrapper (section 3).

## 7. Safety limits

Nothing reaches the library until every check has passed, and a failed restore changes nothing: the target folder is left as it was, and any folder above it that the restore made is removed again, unless something else has put anything in it meanwhile. The Apple code checks one book at a time and builds the library in a hidden folder that is moved into place only when every book has passed.

- A ZIP path that is absolute, contains `..` (or an empty or `.` part), contains a backslash or a drive prefix, or is a link is refused. A repeated entry name is refused.
- Limits (spec 7.4). More than 100,000 files, more than 2 GB unpacked, or a compression ratio over 100 to 1 is refused. Both the sizes a file declares and the bytes that actually come out are checked: declared sizes before any entry of that ZIP is inflated, actual bytes while it is. In a library backup: the 100,000-file limit counts every file in the backup, including those inside its books; the 2 GB limit (2,000,000,000 bytes) applies to each book on its own, and the backup as a whole has no size limit; the ratio is checked for each book (the files inside it against the size of its `.rolltop` file) and for the backup as a whole, in two separate totals, each against the size of the backup file: (a) `library.json` plus the book files as extracted, and (b) `library.json` plus every file inside every book. Each ratio rule applies only to a book or total that unpacks to more than 1,000,000 bytes; the size limits still apply to small ones.
- **Largest single file** (spec 7.4). No single file inside a book archive or a backup may be larger than 50 MB, meaning 50,000,000 bytes: this covers `manifest.json`, `library.json` and every JSON file as well as attachment files. The one exception is a book archive inside a backup (`books/<book id>.rolltop`), which is held only to the book limits above. As with the other limits, both the size a file declares and the bytes that actually come out are checked. A writer must not produce a file over this limit (spec 7.3): the Apple code refuses to write a book archive or a backup in which any book would need one, names the book and the file, and writes nothing (an earlier file at the destination is kept); an automatic backup stops the same way.
- **Not content** (spec 7.5). Two things in a ZIP are not content and do not make a file refused. Bytes that belong to no file (for example, padding between entries) are ignored. Folder entries are accepted only when they are folders of the layout: in a book `manuscript/`, `manuscript/scenes/`, `bible/` and `attachments/`; in a library backup `books/`. Any other folder entry is refused as content the reader does not know. A folder entry's name is checked like any other path (absolute, `..`, links), and each folder entry counts toward the 100,000-file limit. A folder entry's name ends in `/` (`bible/` for the folder `bible`); an entry marked as a folder without that ending is refused.
- **A malformed ZIP directory** (spec 7.4). A ZIP in which two entries share or overlap their headers (an entry's local header and its data must not overlap another entry's), or in which the two copies of an entry's name (in its own local header and in the central directory) differ, is refused. No ZIP written by the app or by ordinary tools is like this. The Apple code also refuses, the same way: a directory whose records or local headers are not where it says or cannot be read whole; an entry whose copies of its sizes disagree (a data descriptor's against the central directory's, which is what the reader reads by); and an entry whose data cannot be read whole where it lies (it runs into the next entry, into the central directory, or past the end of the file).
- **Name length.** No entry name may be longer than 267 bytes, the longest the layout can have (`attachments/` and an attachment's stored name of at most 255 bytes). A longer name is refused as content the reader does not know, while the directory is read.
- **Order of the checks** (spec 7.4). For each ZIP, its count, size and directory checks run before anything inside that ZIP is read, so a ZIP that breaks one of them is refused for that reason even if it was also made by a newer format version. A backup is read in order: its own checks, then `library.json`, then each book in turn, so a fault inside a book is found after the backup's own version has been read.
- Every checksum in the manifest must match, in both directions: every listed file must be present, and every file present must be listed. An attachment file's bytes must hash to the `sha256` in its name and record.
- Every reference inside the book must resolve to a record in the file (a record in the Trash counts). Checked: every node's `parentId`; every scene file's `nodeId` (it must be a scene node of the book, and equal the file name); every run's `entryId` (it must be the id of an entry in `bible/entries.json`, which is empty in milestone 1, so any `entryId` is refused); every attachment record's file; inside a backup, the book's shelf, which must be one of the shelves in `library.json`. Every record's `bookId` must be the manifest's `bookId`.
- **An item in the Trash whose container is gone.** Sync can leave an item in the Trash whose chapter, part or shelf was deleted forever on another device. Such an item is not a broken reference: it is written to archives and backups and comes back in the Trash. A node with `deletedAt` may have a `parentId` that is not in `nodes.json`; a book with `deletedAt` may have a `shelfId` that is not in `library.json`, and then a writer leaves the `shelf` part out of `book.json`; a reader accepts either. A node or book without `deletedAt` must still resolve. The manuscript rule below still applies to a node whose parent is present.
- The manuscript must follow the rules of section 4.4, trashed nodes included: a part at the top level, a chapter at the top level or in a part, a scene in a chapter. A reader refuses a tree that breaks them.
- `library.json` must hold exactly one shelf of kind `standAlone`.
- No id may repeat: not between shelves, not between books, and not between nodes or attachment records of different books of one backup (a repeat would overwrite another book's record).
- Each record must have its required fields with the right types and enum values. Times and hashes must be in the forms of sections 1 and 4.6. Ids in records are read in any letter case and always written back lower case; ids in `manifest.json`, `book.json`'s `shelf`, and file names must already be lower case. **Open point:** whether a reader should refuse an upper-case id in a record, and the JSON Schema that spec 7.4 asks for, are not settled or written yet.
