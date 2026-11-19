# Manicule

A keyboard-driven bookmark manager for Chromium browsers (Chrome, Brave, Edge and others), built on the browser's own bookmarks. It adds tags, notes, favorites, and search operators. With the optional helper, your tags and notes are also saved as plain Markdown files that a sync tool such as Google Drive, Dropbox or Syncthing can carry between machines. A manicule (☞) is the pointing hand readers drew in book margins to mark a passage in the good old days.

## Install with your AI

Give your AI coding agent the link to this page, https://github.com/bendecastro/manicule-releases, and ask it to install Manicule and the helper for you. The steps below are written so it can follow them exactly. Two steps need you: clicking in the browser, and allowing the helper connection.

## 1. Install the extension

Manicule is not in the Chrome Web Store yet. You load it into the browser from a folder.

1. Download https://github.com/bendecastro/manicule-releases/releases/latest/download/manicule-extension.zip.
2. Unzip it into a folder you keep, for example `Manicule-extension` in your home folder's Documents. The browser loads the extension from that folder, so do not delete it.
3. Open your browser's extensions page (in the browser menu, under **Extensions**) and turn on **Developer mode**.
4. Click **Load unpacked** and choose that folder (the one containing `manifest.json`).
5. Check: Manicule appears in the list with the ID `cihgdfpcmbllmmefmdimhppjapoioiaa`, and Alt+Shift+M opens it.

To update: download the zip again, replace the folder's contents with the new files, then click the reload button on Manicule's card on the extensions page.

## 2. Install the helper (optional)

The helper saves your tags, notes and favorites as Markdown files, by default in `~/Documents/Manicule`. It runs on Linux and macOS and needs [Node.js](https://nodejs.org) 22 or newer.

1. Open Manicule's Settings (press `g` then `,` in the app) and go to **Markdown files and helper**.
2. Click **Set up the helper**, then **Allow helper connection**, and accept the browser's prompt. If Settings asks you to reload Manicule, do so.
3. Run the one-line installer in a terminal. Settings shows it with a **Copy** button; it is:

   ```sh
   curl -fsSL https://github.com/bendecastro/manicule-releases/releases/latest/download/install.sh | sh -s -- --extension-id cihgdfpcmbllmmefmdimhppjapoioiaa
   ```

4. Check: within a few seconds Settings shows **Connected ✓** with the helper's version, and `manicule-helper status` in a terminal lists your browser and the Markdown folder. If `manicule-helper` is not found, the installer printed the line to add to your shell's startup file.

The files the helper writes are described in [FORMAT.md](FORMAT.md).

## Main keyboard shortcuts

| Shortcut | Action |
| --- | --- |
| Alt+Shift+M | Open the full app |
| Alt+Shift+O | Open or close the side panel |
| Alt+Shift+S | Save the current page (or click the toolbar button) |
| `:` | Command palette and search |
| `s` or `/` | Search bookmarks |
| `?` | Show keyboard shortcuts, every key by task |
| `j` / `k` | Move down / up |
| `g` then `u` / `f` / `t` | Go to Unsorted, Favorites, Trash |
| `g` then `,` | Settings |
| `Space` | Select the focused row (keys such as `d` and `f` then act on the selection) |
| `f` / `e` / `d` | Favorite / edit tags and note / delete |
| `u` / `Ctrl+Z` | Undo the last change |

Change the global shortcuts from the extensions page: open **Keyboard shortcuts** there.

## Backup

Open Settings, **Backup**. **Export** saves one file with your tags, notes, favorites, smart collections and settings; keep it as your backup. **Import** reads that file back, and also imports from other bookmark managers and browsers. It shows what will change before changing anything. Your bookmarks themselves stay in the browser and its own sync.

License: MIT.
