# Слухайко — notes for Claude

Offline PWA audiobook player for files saved from Telegram (iPhone Safari / Android Chrome).
No build, no dependencies, no tests. UI text and README are in Ukrainian; code comments in English.
Deployed to GitHub Pages from `main` (origin: github.com/lSeverl/slukhaiko).

## Files
- `index.html` — the whole app: CSS → markup → one IIFE script (+ small SW-registration script).
- `sw.js` — offline cache. Navigation is network-first (index.html updates arrive without a bump);
  everything else cache-first. Bump `VERSION` on every release anyway (README rule),
  together with `APP_VERSION` in `index.html` (shown next to the title) — keep them equal.
- `manifest.webmanifest`, `icons/`.

## Script layout (index.html)
- **storage**: `ls` = localStorage wrapper, prefix `slukh.` (keys `rate`, `cur`).
  IndexedDB `slukhayko` v1, stores `files` (Blob, or `{buf,type}` fallback for old Safari) and
  `meta` (keyPath `id`). If IndexedDB fails, falls back to in-memory Maps.
- **item meta**: `{id, name, size, pos, duration, done, order, added}` + optional tag fields
  `{title, artist, album, series, seriesNo, year, track, desc, cover, coverRemote, tags, manual, fileName}`. One flat playlist sorted by `order`.
  `cover` = key `cv:<hash>` of an image Blob in the `files` store, shared between files; deleted when unreferenced.
  Adding optional fields needs no version bump (items with `tags < TAGS_VER` are re-read by `backfillTags()` on boot).
  `manual: true` = user saved info in the «Про книгу» sheet; tags never overwrite it. `fileName` = original name.
  Changing this schema requires `indexedDB.open('slukhayko', 2)` + migration in `onupgradeneeded` —
  users already have data on their phones.
- **render**: `renderList()` rebuilds the list; `refreshRow(it)` updates one row's bar/meta in place;
  `itemView(it)` computes `{pct, meta}`; `renderNow()`, `updateTime()`, `renderRate()`.
- **playback**: `select(id, autoplay)` → blob URL, `pendingSeek` = saved pos − 2 s, applied on
  `loadedmetadata`. `saveNow()` every 5 s from `timeupdate`, on pause, `visibilitychange`, `pagehide`.
  `ended` → mark done, go to next not-done item. `goto(delta)` keeps current play/pause state.
- **tags**: `parseTags(blob)` reads ID3v2.2–2.4 (cp1251/UTF-8 heuristics for "Latin-1" text), MP4 `ilst`
  (moov may be at the end), ID3v1 fallback. Titles equal to the file name and `JUNK_COVERS` (audio-joiner logo) are dropped. `applyTags(it, blob)` fills meta. Series = TXXX SERIES / MVNM / TIT1, ©mvn / ©grp.
- **interruptions**: a `pause` we didn't cause (`appPause` flag, set by `pauseApp()`/`select`) marks `interrupted`;
  the next `play` (system auto-resume or user) calls `reloadAudio(back)` — reassigns `src` and seeks. On-screen Play
  (`play(true)`) reloads too (Android Chrome sometimes resumes silently after an air-raid-alert app takes audio focus).
  Headset/lock-screen Play must NOT reload: measured via adb, a reload destroys Chrome's media session for ~0.7 s.
  `sendPosition()` runs on state changes only, not on `timeupdate`.
  Always pause through `pauseApp()`, never bare `audio.pause()`.
- **media session**: lock-screen controls (−15 / +30 s, prev/next track, seekto).
- **sleep timer**: `checkSleep()` compares wall clock to `sleepEnd`; called from `setInterval`,
  `timeupdate` and `visibilitychange` (background timers are throttled on locked phones). Mode `end` = stop after file.
- **playlist ops**: drag by `.grip` (`startDrag`/`dragStep`/`finishDrag`, pointer events, live DOM reorder;
  `renderList()` is a no-op while `drag` is set), arrow keys on the grip → `move`. `#sortSel` one-shot sort (`SORTS`,
  parts of a book stay together via `bookCmp`). `move`, `removeItems(ids)` / `removeItem(id)`, «Видалити прослухані» button (`#clearDone`, two-tap confirm).
- **book info sheet** (`#info` dialog, `openInfo`/`runSearch`/`pick`/`saveInfo`): query = `translitWord` guess from the
  file name; searches uk.wikipedia (+ Wikidata for author/year/series) and Google Books (keyless, often 429 — ignored).
  Cover is fetched to a Blob when CORS allows, else kept as `coverRemote` URL. «All files of this book» uses `sameBook`.
  sw.js never caches cross-origin API calls (only fonts and images).
- **install**: `beforeinstallprompt` → «Встановити» button in `#installHint`; iOS/other Android get manual instructions.
  Dismissal stored in `ls` key `hintOff`.
- **add files**: sorted by name with `localeCompare('uk', {numeric:true})`, duration probed via a separate `Audio`.

## Checking changes
Syntax: extract the first `<script>` and run `node --check`. Real check: `npx serve .` and open on a phone.
