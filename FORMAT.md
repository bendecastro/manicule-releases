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
| `link_status` | `ok`, `redirect`, `blocked`, `broken`, `unreachable`, `timeout` or `local` | Only after a broken-link check: what it found. `ok`: the page answered. `redirect`: it answered after a redirect to another page. `blocked`: the site refused the checker (401, 403 or 429), which does not mean the page is gone. `broken`: another 4xx or 5xx answer. `unreachable`: no answer (name lookup, TLS or connection failure). `timeout`: no answer in time. `local`: an address on the local network, not checked. |
| `link_http_status` | number | Only after a check that got an HTTP answer: its status code. |
| `link_checked` | time | Only after a check: when it ran. A check does not change `updated`. |
| `manicule` | number, required | The format version, `1`. Files written before Manicule was renamed carry `shelfmark: 1` instead and are still read; Manicule rewrites them with `manicule:`. |

Keys Manicule does not know are kept as they are when Manicule rewrites the file, so you can add your own. They are not stored in the browser, so they do not reach the Manicule export file or another computer except through the files themselves.

The browser's own bookmark ids and folder ids are never in the files; they differ on every computer.

## The body

Everything after the closing `---` is the note, in Markdown. Manicule writes one empty line between the front matter and the note; a record without a note ends at the closing `---`.

## Conflict copies

A sync app such as Syncthing that finds a file changed on two computers at once keeps both, naming one copy like `example-com-3f2a9c01be.sync-conflict-20261114-101010-ABCDEFG.md`, next to the file (also in `.trash/`). Manicule lists these copies in its **Sync conflicts** view, where you choose which version to keep. It never writes, moves or deletes a conflict copy; delete it yourself once you have chosen.

## How Manicule reads changes back

Manicule watches the folder. When a file changes outside Manicule, it reads the file and compares it with what it has:

- If Manicule has no record of that page yet, the file adds one. It is attached to a browser bookmark of the same page when there is one; it does not create a bookmark.
- If they are the same, nothing happens.
- If the file's `updated` is later than Manicule's, the file wins: the whole record takes the file's values.
- Otherwise Manicule keeps its own version, and the file is replaced with it the next time Manicule saves that page.

A file that cannot be read (broken front matter, an unknown format version, an `updated` that is not a time, or a name that does not match its URL) is listed in Settings and changes nothing in Manicule. The file itself is not kept: the next time Manicule saves that page (for example when that bookmark changes in Manicule), it replaces the file with its own version, and the unreadable edit is lost. Fix the file before then to keep your edit.

## What is safe to edit

- **Safe:** `tags`, `favorite`, `excerpt`, `cover` and the note. Set `updated` to the current time (UTC) whenever you change something, or Manicule's older version wins and overwrites your change.
- **Safe:** adding keys of your own.
- **Keep the file readable:** a file Manicule cannot read (see above) is replaced the next time Manicule saves that page, edit and all. Check the front matter is still valid YAML after editing it by hand.
- **Leave alone:** `url`, `normalized_url` and the file name, which identify the page; `locations`, `created`, `deleted_at`, the `link_` keys and `manicule`, which Manicule keeps up to date. To rename, move or delete a bookmark, do it in the browser or in Manicule.
- **Avoid:** creating or deleting files by hand. Bookmarks themselves live in the browser: a new file adds no bookmark, and deleting a file deletes no bookmark.

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
