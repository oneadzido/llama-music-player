# Llama Music Player v1.3.9

**Release date:** Nov 02, 2026

---

### Overview

The Metadata Editor's file-targeting is hardened in this release. A
metadata write now lands on the file the user is editing, whichever
entry path the editor was opened through and whichever file the SAF
picker selected during the session. The selected content URI is the
writer's authoritative target; the absolute path the database and the
playback consumers key on is either resolved from that URI or accepted
from the caller only when the database confirms the two describe the
same file. An unresolved URI from the SAF picker refines rather than
replaces an existing path. The MODIFY action prefers the URI the
current session selected over a cached URI stored for a previously-
bound song, and persists the current session's URI against the current
path when it uses it. The save path captures the target URI, the target
path, and the album-art-removed flag before dispatching the write to
the executor, so a later picker result or a later change to a mutable
Activity field cannot redirect an in-progress save. The success handler
builds the announcement from the submitted update overlaid on the pre-
edit bundle, and the post-write reread is verification only: a reread
failure cannot suppress the `METADATA_UPDATED` broadcast. The broadcast
always carries the saved URI; the path is carried when the editor knows
one, and consumers that key by path resolve it from the URI through
`SongDatabase`. `MusicService` now owns its own `METADATA_UPDATED`
receiver, so an edit lands on the in-memory playlist even when no
activity is alive. Every path that resolves a song's identity for the
database or the broadcast derives its path from the save target rather
than from the Activity's mutable `mFilePath`.

The read side of the editor is hardened in the same release. The
initial load of the current song is performed by a helper that returns
a metadata bundle for every input, so a cached URI whose SAF grant has
been revoked, a supplied URI whose file has moved, and a file whose
embedded tags no longer parse each resolve to the editor's normal
view mode rather than to a crash on open. The URI's write permission
is checked before the attempt, so a grant revoked from Settings → App
Info → Storage is not retried on every open; the file path is then
read directly. When neither source yields a successful read, the
editor stays on the current song in view mode: the fields are
populated from the song database's row for the path, the header stays
on VIEW MODE, and MODIFY is enabled so the user can re-establish file
access through the storage access framework. `onCreate()` wraps the
two calls that touch a cached URI and an unparsed file, so a corrupt
database row or a file whose metadata cannot be read produces a null
field rather than an exception. The editor continues to load the song
the service is actually on: the intent extras name
`MusicService.getCurrentSong()`, whose playlist entry is refreshed by
`refreshMetadataForSong` when the `METADATA_UPDATED` broadcast lands,
so a close and re-open after a save shows the values the service now
carries.

---

### Fixed

- **The editor does not crash when it opens with a stale URI** — A
  cached URI whose SAF grant has been revoked, or a supplied URI
  whose file has moved, used to crash the editor before the user
  could interact with it. The initial read is now performed by
  `readMetadataSafely()`, which returns the editor's normal
  `FILE_UNREADABLE` bundle on every failure. The failure resolves
  to the current song in view mode: the fields are populated from
  the song database's row for the path, the header stays on VIEW
  MODE, and MODIFY is enabled so the user can re-establish file
  access. Revoking SAF permissions from Settings → App Info →
  Storage is the common trigger.

- **The initial read falls back to the path when the URI is
  unreadable** — The URI is tried first because it is the editor's
  write identity; the write permission is checked before the
  attempt, so a grant revoked from Settings is not retried on
  every open. When the URI read does not return `SUCCESS`, the
  file path is tried. When neither source returns `SUCCESS`, the
  editor stays on the current song in view mode with MODIFY
  enabled; pressing MODIFY re-establishes file access through the
  storage access framework.

- **A malformed cached URI and an unparseable file do not crash
  the open** — `onCreate()` wraps `Song.getContentUri()` and
  `CharacterMapper.getSongFromFile()` in try/catch, so a corrupt
  database row or a file whose metadata cannot be read resolves to
  a null field rather than to an exception. The safe read then
  treats the missing field the same way it treats any other absent
  source.

- **Metadata writes land on the file the user is editing** — The
  editor treats the URI the current session selected as the write
  target and derives the absolute path from that URI. When the
  editor is opened with a `song_uri` extra, a supplied `song_path`
  is accepted only when `getFilePathFromUri` resolves the URI to it,
  or when `SongDatabase.getSong(suppliedPath)` returns a row whose
  `getUri()` equals the supplied URI; otherwise `mFilePath` stays
  null. When the editor is opened with only a `song_path`, the URI
  is read from the database as before.

- **URI authority for writing** — `performSaveChanges` dispatches
  the write with the session-selected URI as its target. Every
  later use of the song's identity — the database row, the
  broadcast extras, the failure prompt — is derived from that URI,
  never from a second path lookup that could have moved on.

- **An unresolved URI refines rather than replaces an existing
  path** — When the editor is opened with a `song_uri` extra whose
  URI cannot be resolved to a filesystem path, `mFilePath` is left
  null unless the database confirms a path is paired with that
  exact URI. The SAF picker result augments identity rather than
  replacing it: the pick adds the URI, and a resolved path refines
  `mFilePath` but never removes one the caller already supplied.
  The URI is authoritative for writing; the path is authoritative
  for matching receivers.

- **MODIFY prefers the current session's URI over a cached URI** —
  The MODIFY action prefers the session-selected URI when it
  carries write permission, persists it against the current path
  through `saveUriToDatabase()`, and only falls back to the stored
  URI when the session has none.

- **In-flight saves are pinned to the target captured at
  dispatch** — `performSaveChanges` captures the target URI, the
  target path, the album-art-removed flag, the pre-edit bundle, and
  the submitted update into `final` locals before it dispatches the
  write to the executor, and passes them to the success and failure
  handlers as parameters. A later SAF pick or a later change to
  `mGrantedFileUri` or `mFilePath` cannot redirect a save that is
  already running.

- **Database rows identify the song that was written** —
  `onSaveSuccess` uses the captured target path for the database
  row, falling back to a path resolved from the saved URI when the
  captured path is null. A path that cannot be derived from either
  source leaves the database row unwritten rather than writing a
  row against the Activity's mutable path.

- **The post-write reread is verification, not a gate** —
  `onSaveSuccess` builds the announcement from the submitted
  update overlaid on the pre-edit bundle through
  `buildAnnouncementFromSubmitted`. The post-write reread supplies
  values when it succeeds; when it fails, the submitted values are
  announced anyway, because the write itself succeeded. A reread
  that returns `FILE_UNREADABLE` or `PARSE_FAILED` cannot suppress
  the `METADATA_UPDATED` broadcast, cannot prevent the database
  row from being written, and cannot block playback resumption.

- **Broadcasts carry the URI and, when known, the path** — The
  `METADATA_UPDATED` broadcast always carries the saved URI. The
  path is present when the editor knows one. Receivers that key by
  path resolve it from the URI through `SongDatabase` in their
  `resolveBroadcastSongPath` helper.

- **SAF-only writes reach path-keyed consumers** — `MusicPlayer`,
  `PlaylistActivity`, and `MusicService` each have a
  `resolveBroadcastSongPath` helper that resolves a URI-only
  broadcast to a path by matching against `SongDatabase`. On a
  SAF-only write, the editor knows the URI but not the path; the
  helper lets path-keyed consumers — the player's current-song
  check, the playlist row lookup, the service's in-memory playlist
  refresh — still find the edited song.

- **MusicService owns its own METADATA_UPDATED receiver** —
  `MusicService` registers a receiver for `METADATA_UPDATED` in
  `onCreate` and unregisters it in `onDestroy`, so the edit lands
  on the in-memory playlist for the service's whole lifetime —
  including the case where the player activity has been reclaimed
  but playback continues.

- **Shared album-art cache evicted under both keys** —
  `MetadataEditor` evicts the shared album-art cache for both the
  path and the URI before the broadcast.
  `MusicService.refreshMetadataForSong` evicts both keys, then
  stores the resolved art under both keys, so every consumer —
  whether it keys by path (the player, the carousel) or by URI —
  sees the new art on its next access.

- **Failure handling uses the failed target** — `onSaveFailure`
  takes the captured target path as a parameter and, on
  `PERMISSION_DENIED`, writes the fallback database row against
  that path instead of `mFilePath`.

---

### Under the Hood

- `MetadataEditor.loadMetadataAsync()` — Delegates the read to
  `readMetadataSafely()`. The overlay show/hide is unchanged. The
  status branch routes every non-`SUCCESS` outcome —
  `FILE_UNREADABLE`, `PARSE_FAILED`, and the unsupported-format
  case — through `loadCurrentSongInViewMode()`, so the editor
  stays on the current song in view mode with MODIFY enabled
  instead of entering the "no file" state or disabling the action
  button.

- `MetadataEditor.readMetadataSafely()` — New helper. Checks the
  URI's write permission first; a grant that has been revoked
  from Settings → App Info → Storage is cleared rather than
  retried, so the URI read is skipped and the path is read
  directly. Then tries the URI, then the path. Returns a bundle
  for every input: a URI read that returns `SUCCESS`
  short-circuits and, when no path is known yet, resolves one
  from the URI so a later write has both identities. When the URI
  read does not return `SUCCESS`, the path is tried. When neither
  source returns `SUCCESS`, the URI's bundle is returned when one
  exists — it carries the status and format that match the
  identity the editor was opened with — and a constructed
  `FILE_UNREADABLE` bundle otherwise. Never throws.

- `MetadataEditor.loadCurrentSongInViewMode()` — New helper.
  Called from the `FILE_UNREADABLE`, `PARSE_FAILED`, and
  unsupported-format branches of both failure paths — the
  initial `loadMetadataAsync()` read and the `onActivityResult()`
  re-read after a SAF pick. Populates the title, artist, album,
  and genre fields from the song database's row for `mFilePath`,
  restores the header to VIEW MODE, sets the action button to
  MODIFY, and enables it. The editor's "no file" state is left to
  the `awaiting_file_selection` entry path and is never reached
  through a read failure.

- `MetadataEditor.onCreate(Bundle)` — Wraps
  `Song.getContentUri()` and `CharacterMapper.getSongFromFile()`
  in try/catch, so a malformed cached URI or an unparseable file
  resolves to a null field rather than to an exception. The rest
  of the method is unchanged. When the intent carries a
  `song_uri` extra, the URI is the editor's write target. A
  supplied `song_path` is accepted only when
  `getFilePathFromUri(mGrantedFileUri)` resolves to it, or when
  `SongDatabase.getSong(suppliedPath)` returns a row whose
  `getUri()` equals the supplied URI. Otherwise `mFilePath` stays
  null. When the intent carries only a `song_path`, the URI is
  read from the database as before.

- `MetadataEditor.handleModifyClick()` — MODIFY prefers
  `mGrantedFileUri` when the session-selected URI has write
  permission; the stored URI is consulted only when the session
  has none. When the session-selected URI is used,
  `saveUriToDatabase()` persists it against `mFilePath` before
  edit mode is entered.

- `MetadataEditor.onActivityResult(int, int, Intent)` — The SAF
  picker branch sets `mGrantedFileUri = selectedUri`, then
  overwrites `mFilePath` only when
  `getFilePathFromUri(selectedUri)` resolves to a non-null path.
  A pick whose URI has no filesystem path preserves the existing
  `mFilePath` rather than clearing it. When the re-read that
  follows the pick does not return `SUCCESS`, the branch routes
  through `loadCurrentSongInViewMode()`.

- `MetadataEditor.performSaveChanges()` — Captures
  `final Uri targetUri = mGrantedFileUri`,
  `final String targetPath = mFilePath`,
  `final boolean albumArtWasDeleted = mAlbumArtDeleted`,
  `final MetadataReader.MetadataBundle preEditBundle = mOriginalData`,
  and `final MetadataWriter.UpdateBundle submittedUpdate = update`
  before the executor submission. The success and failure handlers
  take these as parameters. The write itself is dispatched with
  `targetUri`.

- `MetadataEditor.onSaveSuccess(Uri, String, boolean,
  MetadataBundle, UpdateBundle)` — Builds the announcement from
  the submitted update overlaid on the pre-edit bundle via
  `buildAnnouncementFromSubmitted`. The post-write reread supplies
  values when it succeeds. `mOriginalData` is set to the verified
  bundle when the reread succeeds, and to the announcement
  otherwise. The database row and broadcast path come from
  `savedPath` when it is non-null, and from
  `getFilePathFromUri(savedUri)` otherwise. The broadcast always
  carries the URI; the path is included when known. The shared
  album-art cache is evicted for both keys before the broadcast.

- `MetadataEditor.buildAnnouncementFromSubmitted(MetadataBundle,
  UpdateBundle)` — New helper. Overlays the submitted update on
  the pre-edit bundle. A null field in the update inherits the
  pre-edit value; a zero-length `albumArt` array means "removed"
  and becomes a null album art on the bundle. A null pre-edit
  bundle is handled: only the update's non-null fields appear.

- `MetadataEditor.evictAlbumArtCaches(String, Uri)` — New helper.
  Evicts the shared album-art cache for both the path key and the
  URI key before the broadcast.

- `MetadataEditor.onSaveFailure(WriteResult, String)` — Takes
  `failedTargetPath` as a parameter and uses it for the
  `PERMISSION_DENIED` database repair row.

- `MetadataEditor.saveUriToDatabase()` — Called from
  `handleModifyClick()` when the session-selected URI is used, in
  addition to its previous call sites.

- `MusicPlayer.resolveBroadcastSongPath(Intent)` — New helper.
  Returns the broadcast's `song_path` extra when present;
  otherwise resolves the `song_uri` extra against `SongDatabase`
  and returns the matching song's path.

- `MusicPlayer.mMetadataUpdatedReceiver.onReceive` — Uses
  `resolveBroadcastSongPath` to resolve the edited song's path,
  then calls `MusicService.refreshMetadataForSong` with the
  submitted values and evicts the carousel's per-path cache via
  `AlbumArtCarousel.invalidatePath`.

- `PlaylistActivity.resolveBroadcastSongPath(Intent)` — New
  helper, same shape as `MusicPlayer`'s.

- `PlaylistActivity.mMetadataUpdatedReceiver.onReceive` — Uses
  `resolveBroadcastSongPath` to resolve the edited song's path,
  then updates the display row and, if the edited song is the
  current playing song, refreshes the playing indicator.

- `MusicService.ACTION_METADATA_UPDATED` — New broadcast-action
  constant alongside the existing `ACTION_*` constants.

- `MusicService.registerMetadataUpdatedReceiver()` — New method,
  called from `onCreate` and torn down in `onDestroy`. Registers
  a receiver for `METADATA_UPDATED` that uses
  `resolveBroadcastSongPath` and calls `refreshMetadataForSong`.

- `MusicService.resolveBroadcastSongPath(Intent)` — New helper,
  same shape as `MusicPlayer`'s and `PlaylistActivity`'s.

- `MusicService.refreshMetadataForSong(...)` — Evicts the shared
  album-art cache under both the path key and the URI key, then
  stores the resolved art under both keys. Re-announces
  unconditionally through `broadcastUpdate()`, which commits the
  notification on the same main-thread turn.

---

## System Requirements

- **Minimum SDK:** Android 6.0 (API 23)
- **Target SDK:** Android 15 (API 35)
- **Recommended RAM:** 1 GB or higher

---

## Download

Download the APK from the Assets section of this release.

---

## Installation

1. Download the APK file
2. Enable "Unknown Sources" in your device settings
3. Open the APK file and tap "Install"

---

## First Time Setup

1. Tap the MANAGER button on the main screen
2. Tap IMPORT FOLDER to select your music folder
3. Tap UPDATE to save your selection
4. The app will scan your folder and build the playlist

---

## How to Use

| Feature | How to Access |
|---------|---------------|
| Equalizer | Tap EQUALIZER button on main screen |
| Metadata Editor | Tap METADATA button on main screen |
| Folder Manager | Tap MANAGER button on main screen |
| Playlist | Tap PLAYLIST button on main screen |
| Themes | Tap LLAMA button on main screen, then THEMES |
| Language | Tap LLAMA button on main screen, then the language button in the version header |
| Volume gesture | Swipe up or down along the album-art edge |
| Previous / next song | Swipe left or right across the album art, or tap the transport buttons |
| Playlist fast-scroll | Drag vertically along the left or right edge of the playlist |
| Sort mode | Tap SORT in the playlist header |

---

## Credits

- jaudiotagger by jthink Ltd
- FontAwesome by Fonticons, Inc.

---

## License

Licensed under the MIT License

Copyright (c) 2026 Richard Korbla Adzido

Made with love in Ghana