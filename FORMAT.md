# Manicule's Markdown files

With the helper installed, Manicule keeps one Markdown file for every bookmarked page in a folder on your computer (by default `Manicule` in your Documents folder; change it in Manicule's settings or with `manicule-helper folder <path>`). You, a sync app or an AI tool can read and edit these files; Manicule reads the changes back.

This page describes format version 1. The code that writes and reads it is `writeSidecarFile` and `readSidecarFile` in `packages/core/src/sidecar-file.ts` in the Manicule repository; this page and that code must agree.

## One file per page

- Each file stands for one page, identified by its **normalized URL** (the URL with tracking parameters, `www.`, the `#fragment` and similar differences removed). Several browser bookmarks of the same page share one file.
- A file has YAML front matter between two `---` lines, then the note as the body.
- Files are UTF-8, with `\n` line endings. Manicule writes a file at most 1,000,000 bytes long; a bigger record is not written and Settings says so.
- Writes are atomic (a temporary file, then a rename), so a sync app or an editor never sees half a file.

## File names

The name is a readable slug of the normalized URL plus a hash, for example `example-com-articles-postgres-indexing-3f2a9c01be.md`:

1. Take the normalized URL without its scheme (`https://`).
2. Remove accents, lower-case it, and replace every run of characters other than `a`–`z` and `0`–`9` with `-`; trim `-` at both ends; keep at most 60 characters (and trim a `-` left at the end). An empty result becomes `bookmark`.
3. Add `-` and the first 10 hex digits of the FNV-1a 64-bit hash of the normalized URL's UTF-8 bytes, then `.md`.

The hash keeps two URLs apart when their slugs are the same. A file whose name does not match its own `url` is reported in Settings and not read in. Other files in the folder (any name that is not of this form) are left alone.

## The `.trash/` folder

When the last browser bookmark of a page is removed, its record goes to Manicule's Trash and its file moves to `.trash/` inside the folder, with the same name and a `deleted_at` time. Restoring it moves the file back. After 30 days in the Trash the record and its file are deleted.

## Front matter

Keys are written in this order. Times are ISO 8601 in UTC, for example `2026-10-25T09:14:00Z`. Strings are written plain when that cannot be misread, otherwise double-quoted with JSON escapes.

| Key | Type | Meaning |
| --- | --- | --- |
| `url` | string, required | The page's address as it was saved. |
| `normalized_url` | string | The URL Manicule matches on (see above). When missing, it is computed from `url`. |
| `title` | string | The bookmark's title. Empty when missing. |
| `locations` | list of maps | Where the page's browser bookmarks are; for a file in `.trash/`, where they were, used to put them back. `[]` when none. Each entry has `root`, `syncing` and `path`. |
| `root` | `bookmarks-bar`, `other`, `mobile` or `managed` | Inside a `locations` entry: the browser's top folder (Bookmarks bar, Other bookmarks, Mobile bookmarks, or folders managed by an organisation), never a translated folder title. |
| `syncing` | `true` or `false` | Inside a `locations` entry: whether that top folder is the one the browser syncs to an account. |
| `path` | list of strings | Inside a `locations` entry: the titles of the folders below `root`, outermost first. A hint: folder titles can repeat or change. |
| `tags` | list of strings | The page's tags, for example `[postgres, performance]`. |
| `favorite` | `true` or `false` | Whether the page is a favorite. |
| `excerpt` | string or `null` | A short description of the page. |
| `cover` | string or `null` | The address of the page's cover image. |
| `created` | time, required | When the record was first created. |
| `updated` | time, required | When the record last changed. Decides which version wins when a file and Manicule disagree (below). |
| `deleted_at` | time | Only in `.trash/`: when the record went to the Trash. |
| `link_status` | `ok`, `redirect`, `blocked`, `broken`, `unreachable`, `timeout` or `local` | Only after a broken-link check: what it found. `ok`: the page answered. `redirect` with no `link_http_status`: the page answered with a redirect, which the checker did not follow, so the destination was not checked; with a `link_http_status` (checks made before the checker stopped at redirects): it answered after a redirect to another page. `blocked`: the site refused the checker (401, 403 or 429), which does not mean the page is gone. `broken`: another 4xx or 5xx answer. `unreachable`: no answer (name lookup, TLS or connection failure). `timeout`: no answer in time. `local`: an address on the local network, not checked. |
| `link_http_status` | number | Only after a check that got an HTTP answer: its status code. |
| `link_checked` | time | Only after a check: when it ran. A check does not change `updated`. |
| `type` | `article`, `video`, `image` or `link` | The page's media type, detected when it was saved (a newer Manicule may write other values, which are kept). A file without it gets its type from the address alone (a YouTube or Vimeo watch page is `video`, a `.jpg` is `image`, the rest `link`). |
| `type_chosen` | `true` | Written only when you chose the type in Manicule; detection then never changes it. |
| `facts` | map | Facts about the page saved with it, by type: article `author`, `site`, `published`, `word_count`; video `channel`, `duration` and `position` (seconds), `uploaded`, `chapters` (a list of maps with `at`, in seconds, and `title`); image `width`, `height`, `source_page`, `image_url`, `colors` (`#rrggbb`). A value of the wrong kind is ignored. |
| `manicule` | number, required | The format version, `1`. Files written before Manicule was renamed carry `shelfmark: 1` instead and are still read; Manicule rewrites them with `manicule:`. |

Keys Manicule does not know are kept as they are, so you can add your own: Manicule stores them with the record in the browser, writes them back when it rewrites the file, and carries them in the Manicule export file. A newer Manicule's keys survive an older one the same way ([Fields and compatibility](#fields-and-compatibility)).

The browser's own bookmark ids and folder ids are never in the files; they differ on every computer.

## The body

Everything after the closing `---` is the note, in Markdown. Manicule writes one empty line between the front matter and the note; a record without a note ends at the closing `---`.

From stage 3 the body can end with a `## Highlights` section, which is not part of the note ([Highlights](#highlights)).

## Conflict copies

A sync app such as Syncthing that finds a file changed on two computers at once keeps both, naming one copy like `example-com-3f2a9c01be.sync-conflict-20261114-101010-ABCDEFG.md`, next to the file (also in `.trash/`). Manicule lists these copies in its **Sync conflicts** view, where you choose which version to keep. It never changes, moves or deletes a conflict copy; delete it yourself once you have chosen. Manicule makes one kind itself: a file it could not read, kept before Manicule replaces it (see below), named like `example-com-3f2a9c01be.sync-conflict-20261114-101010-MANICUL.md`.

## How Manicule reads changes back

Manicule watches the folder. When a file changes outside Manicule, it reads the file and compares it with what it has:

- If Manicule has no record of that page yet, the file adds one. It is attached to a browser bookmark of the same page when there is one; it does not create a bookmark.
- If they are the same, nothing happens.
- If the file's `updated` is later than Manicule's, the file wins: the whole record takes the file's values.
- Otherwise Manicule keeps its own version, and the file is replaced with it the next time Manicule saves that page.

A file that cannot be read (broken front matter, an unknown format version, an `updated` that is not a time, or a name that does not match its URL) is listed in Settings and changes nothing in Manicule. The file stays as it is until Manicule next saves that page (for example when that bookmark changes in Manicule). Then Manicule first keeps the unreadable file as a conflict copy, which shows in **Sync conflicts**, and replaces the file with its own version. Nothing in the unreadable file is lost when Manicule saves over it: copy your edit back from the conflict copy, fix it, and delete the copy. Saving over the same unreadable text again makes no second copy, and a conflict copy is never overwritten. A helper too old to keep the copy does not replace the file: only that bookmark waits (every other bookmark is still saved), and Settings name the file and say why. Fix the file (Manicule then saves the bookmark to it) or update the helper.

## What is safe to edit

- **Safe:** `tags`, `favorite`, `excerpt`, `cover` and the note. Set `updated` to the current time (UTC) whenever you change something, or Manicule's older version wins and overwrites your change.
- **Safe:** adding keys of your own.
- **Keep the file readable:** a file Manicule cannot read (see above) is replaced the next time Manicule saves that page; your version is kept as a conflict copy in **Sync conflicts**, to copy your edit back from. Check the front matter is still valid YAML after editing it by hand.
- **Leave alone:** `url`, `normalized_url` and the file name, which identify the page; `locations`, `created`, `deleted_at`, the `link_` keys and `manicule`, which Manicule keeps up to date. To rename, move or delete a bookmark, do it in the browser or in Manicule.
- **Avoid:** creating or deleting files by hand. Bookmarks themselves live in the browser: a new file adds no bookmark, and deleting a file deletes no bookmark.

## Fields and compatibility

This section is the one home for every field Manicule will store per page, built or not yet built ([decisions 355–359](decisions/log.md)). A field that is not built yet is **reserved**: its name and meaning are fixed now, and the work item named beside it builds it. A new field takes its row here, or is added here by these rules in the same change that builds it.

Rules:

1. **Names.** Front matter keys are `snake_case`; the same field in the browser's record is `camelCase` (`deleted_at` is `deletedAt`). One name per field; a reserved name is never used for anything else, and a field is never renamed.
2. **Adding a field never changes the version.** `manicule:` changes only for a breaking change: a field whose meaning or type changes, a field that becomes required, or a change to the file name, the body or the layout of the folder. Adding a field, adding a value to a list of values, or adding a file next to the Markdown file is not breaking.
3. **Every version keeps what it does not know.** A front matter key Manicule does not know (a newer Manicule's field, or one of your own) is kept, with its lines exactly as written, through reading the file, the browser's record, sync, the Manicule export file and every rewrite. When the file wins (its `updated` is later), its unknown keys replace the record's. When Manicule's record wins and holds no unknown keys, it takes the file's without changing `updated`; otherwise the record's unknown keys are written.
4. **An older Manicule never deletes a newer one's fields.** It keeps them by rule 3. A file with a newer `manicule:` than it reads is not saved over: that page's file waits and Settings say to update Manicule. (Today a Manicule that meets an unknown version keeps the file as a conflict copy and then saves over it, see [How Manicule reads changes back](#how-manicule-reads-changes-back); no version 2 exists yet, and the waiting behaviour ships before the first build that writes one.)
5. **Reading never needs a field that was added later.** A missing field reads as its empty value (none, `false`, `[]`), so a file written before the field existed is still valid.
6. **Only Manicule's own fields decide anything.** Unknown keys are kept and shown nowhere in the app; search does not read them.

### Every field

Status: **built** = written today. **reserved (item)** = not written yet; the work item named builds it.

| Front matter | Record | Type | Meaning | Status |
| --- | --- | --- | --- | --- |
| `url` | `url` | string | The address as saved. | built |
| `normalized_url` | `normalizedUrl` | string | The address Manicule matches on; it names the file. | built |
| `normalization` | (none) | number | The version of the URL normalization rules that computed `normalized_url`; absent means 1 ([URL normalization version](#url-normalization-version)). | reserved (the first change to the rules) |
| `title` | `title` | string | The bookmark's title. | built |
| `locations` | `chromeNodes` / `restoreKeys` | list of maps | Where the page's bookmarks are, or were while in the Trash. | built |
| `tags` | `tags` | list of strings | The page's tags. | built |
| `favorite` | `favorite` | `true` / `false` | Favorite or not. | built |
| `excerpt` | `excerpt` | string or `null` | A short description. | built |
| `cover` | `cover` | string or `null` | The cover image's address. | built |
| `created` | `created` | time | When the record was first created. | built |
| `updated` | `updated` | time | When a field you set last changed; decides which version wins. | built |
| `deleted_at` | `deletedAt` | time | When the record went to the Trash. | built |
| `link_status`, `link_http_status`, `link_checked` | `linkCheck` | see above | The last broken-link check. | built |
| `type` | `type` | `article`, `video`, `image` or `link` | The media type (decision 244); more values may be added. | built |
| `type_chosen` | `typeChosen` | `true` / `false` | The user chose the type, so detection never changes it. | built |
| `facts` | `facts` | map | Typed facts saved with the page. Article: `author`, `site`, `published` (time), `word_count`. Video: `channel`, `duration` (seconds), `uploaded` (time), `chapters` (list of `{ at, title }`, `at` in seconds), `position` (seconds watched). Image: `width`, `height`, `source_page`, `image_url`, `colors` (list of `#rrggbb`). | built |
| `page_text` | `pageText` | time | When the current page text was taken; absent when there is none. | built |
| `page_text_file` | `copies.text.reference` | `<file> <taken> <stated>` | Which page text file is current: its whole name (`-` for one kept before copy files had their content in the name), when it was taken and when this was stated. One entry, so a rewrite by an older Manicule never pairs one file with another's time. When `page_text` differs from `<taken>`, an older Manicule took a newer copy and `page_text` wins. | built |
| `page_text_cleared` | `copies.text.cleared` | time | Delete page text cleared every page text stated up to this time; written whenever it exists. | built |
| `image_copy` | `imageCopy` | time | When the current image copy or screenshot was taken. | built |
| `image_copy_file`, `image_copy_cleared` | `copies.image` | as `page_text_file`, `page_text_cleared` | The same for the image copy or screenshot. | built |
| `full_copy` | `fullCopy` | time | When the full copy file was last taken; `has:copy` reads it. | reserved (066-full-copies) |
| `later` | `later` | time | Marked Read later, at this time; absent when not marked. | reserved (064-read-later) |
| `up_next` | `upNext` | number | Place in Up next, 1 first; absent when not in Up next. Positions are whole numbers with no gaps. | reserved (064-read-later) |
| `dropped` | `dropped` | time | When the read-later mark lapsed unread (Dropped from queue); cleared when marked again. | reserved (064-read-later) |
| `read` | `read` | time | When the page was last marked read. | reserved (064-read-later) |
| `reading_position` | `readingPosition` | number from 0 to 1 | How far through the page text the reader was left, to resume there. A video's place is `facts.position`. | reserved (064-read-later) |
| `due` | `due` | time | When it is due. Snooze sets a later `due`; Done removes it. There is no separate snooze field: `is:snoozed` is a `due` in the future. | reserved (063-reminders) |
| `remembered` | `remembered` | time | A remembered page, remembered at this time. Kept when the page gets a bookmark (decision 405): a Chrome bookmark of this URL on the reading machine takes precedence, so the page shows there as a bookmark, and as a remembered page wherever no bookmark exists. Cleared when the page is deleted as a bookmark in Manicule or the old address is trashed by an explicit Manicule address change; a rekey carries it to the new address. | reserved (054-remember) |
| `last_used` | `lastUsed` | time | When a remembered page was last used by an explicit action (Remember, re-Remember, restore, a note or tag edit), set in the same write as `updated` and carried with the winning record like any field. Opens and visits never write it; they stay in each machine's usage store, and the expiry date on a machine is the latest of this field, `updated` and local last use (the later of `opens` and permitted `history`), plus the expiry period (decision 405). A clock without `remembered` does not make a page remembered or eligible for expiry. | reserved (054-remember) |
| `ai` | `ai` | map | AI metadata with its own `schema` version ([PRD §8.2](PRD.md#82-sidecar-store-and-fields-owned-by-manicule)). | reserved (010-ai) |
| `manicule` | (none) | number | The format version, `1`. | built |
| (body) | `note` | Markdown | The note: the body up to `## Highlights`. | built |
| (body) `## Highlights` | `highlights` | list | Highlights and clips ([Highlights](#highlights)). | reserved (065-highlights) |
| (none) | `unknownFrontmatter` | list of raw entries | Front matter keys this version does not know, kept by rule 3. | built |

Not stored per page, on purpose: **stacks** (a stack is a folder, and folders have no file: a stack is a synced setting, decision 357), **opens and other usage** (per computer, in the browser only, so `opened:` can differ between computers), embeddings and the search index (rebuilt on each computer), and Chrome's ids.

A reserved field is written only when it has a value. Taking a copy writes `page_text`, `image_copy`, `full_copy` and `facts` without changing `updated`, as a link check does; the other reserved fields change `updated` like any edit.

## Highlights

(Reserved: work item 065-highlights builds it.) Highlights and clips sit in the body under a line that is exactly `## Highlights`. The first such line outside a fenced code block ends the note; everything after it belongs to the section, to the end of the file. Manicule always writes the section last, after one empty line. A note cannot hold its own `## Highlights` heading; any other heading is fine.

Each entry starts with one line that holds its details as JSON in an HTML comment, which Markdown viewers hide, then the quoted text, then an optional note:

```markdown
## Highlights

<!-- manicule:highlight {"id":"h3k9q2","created":"2026-11-22T09:14:00Z","prefix":"most queries, ","suffix":" only when the"} -->
> A partial index helps only when the query's WHERE clause matches it.

Check this against the slow report query.

<!-- manicule:clip {"id":"c81za0","created":"2026-11-22T09:20:00Z"} -->
> **Ingredients:** 200 g flour, 2 eggs.
```

- `manicule:highlight` is a passage marked on the page; `manicule:clip` is a part of the page kept with the bookmark.
- The JSON has `id` (unique within the file), `created` (time) and, for a highlight, the anchor used to draw it again on the live page: `prefix` and `suffix` (up to 32 characters of page text just before and after the passage, as in the W3C text-quote selector) and, when known, `start` (its offset in the page text). Keys Manicule does not know are kept.
- The quoted text is every `> ` line after the comment line; the passage itself is that text without `> `. Lines after it, up to the next entry's comment line, are the entry's note.
- Highlights are your data: editing the quoted text or note by hand is safe; keep each comment line as it is.

## Files next to the Markdown file

From stage 2 a page can have other files beside its Markdown file. Each one's name is the Markdown file's name with `.md` replaced:

| File | Holds | Written by |
| --- | --- | --- |
| `<name>.text.<hash>.md` | The page text: the readable text of the page as Markdown, with front matter `url`, `taken` (time) and `manicule: 1`. At most 1,000,000 bytes; longer text is cut and ends with a line saying so. `<hash>` is the first 16 hex digits of the file's SHA-256, so each capture is its own file that is never changed; `page_text_file` names the current one. | 117-copies-by-content |
| `<name>.image.<hash>.<ext>` | The image copy, or a screenshot when the page has no cover; `<ext>` is `png`, `jpg`, `webp` or `gif`. At most 10 MB. Named and never changed like the page text; `image_copy_file` names the current one. | 117-copies-by-content |
| `<name>.text.md`, `<name>.image.<ext>` | Copies kept before 117-copies-by-content, rewritten in place on each capture. Read for a reference with `-` as its file; no longer written. | 051-save-time-capture |
| `<name>.mhtml` | The full copy: the whole page as one MHTML file. At most 50 MB. | 066-full-copies |

For example `example-com-articles-postgres-indexing-3f2a9c01be.md` has `example-com-articles-postgres-indexing-3f2a9c01be.text.md` and `example-com-articles-postgres-indexing-3f2a9c01be.mhtml`. These files are not records: Manicule never reads them as bookmarks, and editing them changes no bookmark. They follow their Markdown file:

- **Trash and restore:** they move to `.trash/` with it, and back. If a file of the same name is already where one moves to, the one changed last stays and the other is deleted.
- **Purge:** they are deleted with it after 30 days in the Trash.
- **New address:** not renamed today. A bookmark whose address changes so that its normalized URL changes gets a new record; the files beside the old file stay with the old record, go to the Trash with it and are purged with it after 30 days. The new record has none until its next save.
- **Conflict copies:** a sync app's copy of one, such as `<name>.text.sync-conflict-20261114-101010-ABCDEFG.md`, is listed in **Sync conflicts** under its page, and is never changed or deleted by Manicule. A conflict copy of an image copy is not listed: it is left alone in the folder (decision 383).
- **A file with no Markdown file** beside it (for example one still syncing) is left alone.

`<name>` is the Markdown file's name when the copy was taken; a copy file is never renamed. Without the helper, or with a helper too old to write them (one that does not list `copy-files`), the same copies are kept in the browser, with the record, and written once a helper that can write them connects.

## URL normalization version

The normalized URL names every file and matches every bookmark, so a change to the rules that compute it (for example a new tracking parameter to remove) changes some pages' names. The rules have a version, `1` today. A file written under newer rules says so with `normalization:` in its front matter; a file without it was written under version 1. A Manicule with newer rules migrates once, in the browser and in the folder: it computes every record's new normalized URL, merges records that now share one (below), renames their files and the files beside them, and writes `normalization:` into each file. A Manicule that finds a file with a newer `normalization:` than its own rules stops reading and writing the whole folder, and Settings say to update Manicule; it keeps working in the browser alone, because its keys would no longer match the folder's and would make duplicate files.

When two records merge, the one with the later `updated` gives every single-value field; `tags` are joined; both notes are kept, the older one after a line `---`; highlights are joined by `id`; the bookmark links of both stay; `created` is the earlier one. Activity logs the merge, and Undo takes it back.

## Example

```markdown
---
url: https://example.com/articles/postgres-indexing?utm_source=feed
normalized_url: https://example.com/articles/postgres-indexing
title: Postgres indexing in practice
locations:
  - root: bookmarks-bar
    syncing: true
    path: [Databases]
tags: [postgres, performance]
favorite: true
excerpt: A practical guide to B-tree and GIN indexes.
cover: https://example.com/cover.png
created: 2026-10-25T09:14:00Z
updated: 2026-10-25T09:20:00Z
link_status: ok
link_http_status: 200
link_checked: 2026-10-25T10:00:00Z
manicule: 1
---

Read the section on partial indexes again before the migration.
```
