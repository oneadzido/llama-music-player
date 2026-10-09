# Llama Music Player v1.2.7

**Release date:** Oct 9, 2026

## Overview

Three changes ship in this release — removing a folder no longer
wipes the playlist, the reveal never shows the placeholder during its
cross-fade, and the cold-start overlay can no longer be left up by a
library load that never completes.

The songs table holds song paths. The selected-folder set holds
folder URIs. The folder-update cycle's completion callback passed the
folder URIs to a deletion that reconciles against song paths, so no
row's path ever matched a keep entry and every row was removed. The
completion callback no longer deletes songs; the scan that runs
inside the update cycle is the single reconciler of the songs table,
and it operates on song paths, which are the values the table stores.

The reveal — the cross-fade the carousel runs when the incoming
current path is neither of the previous window's neighbours — now
waits for the incoming art to resolve before it begins. While it
waits, the outgoing art stays on screen. The moment the incoming art
resolves, the cross-fade runs from the outgoing art to the incoming
art in one transition. The user never sees the placeholder between
the two.

The cold-start overlay is kept alive by a heartbeat that re-posts
itself while the `LIBRARY_LOAD` operation is on the stack. The
heartbeat now stops after a bounded deadline. When it stops, the
overlay is torn down and the user is told the library could not be
loaded. The service bind's synchronous return value is checked for
the same reason: a refused bind cannot be allowed to leave the
overlay up with no callback ever coming to hide it.

## Fixed

- **Removing a folder no longer wipes the playlist** — The songs
  table holds song paths. The selected-folder set holds folder URIs.
  The folder-update cycle's completion callback passed the folder
  URIs to a deletion that reconciles against song paths, so no row's
  path ever matched a keep entry and every row was removed. The
  callback fired only when the user had staged the Loose Tracks
  entry for removal — the loose clear runs in the same branch — but
  the deletion inside it targeted the songs table, not the loose
  table, so removing any folder under that combination wiped the
  library.

  The completion callback no longer deletes songs. The scan that
  runs inside the folder-update cycle already reconciles the songs
  table against the paths it found under the kept folders. When the
  user has staged the Loose Tracks entry for removal, the
  loose-songs table is cleared before the scan runs, so the scan's
  reconciliation leaves the songs table holding exactly the songs
  under the kept folders.

  Removing a folder now removes the songs under that folder and
  leaves every other folder's songs and every loose track alone.
  Removing the Loose Tracks entry removes the loose tracks and
  leaves folder songs alone.

- **The reveal never shows the placeholder mid-transition** — The
  reveal is a cross-fade: the outgoing art fades out as the incoming
  art fades in, with a small overlap. For that shape to hold, both
  bitmaps must be available on the frame the cross-fade begins. They
  were not always both available. A file opened from another app is
  pre-decoded in parallel with the playlist rebuild that produces
  the reveal's window, and whichever finished last was the one the
  reveal waited on. When the preload had not finished, the incoming
  path's bitmap was not in the cache, and the cross-fade ran on the
  placeholder. When the preload resolved a moment later, the
  carousel swapped the overlay's bitmap from placeholder to real
  art mid-fade. The user saw three states — old art, placeholder,
  new art — for what should have been one transition.

  The carousel now gates the reveal on the incoming path's art being
  *resolved*. Resolved means one of two things: the path's bitmap is
  in the shared cache, or the path's decode has completed and
  produced no bitmap because the file has no embedded art. Until one
  of those is true, the carousel enters a pending state: the
  outgoing art stays on screen, no overlay is created, and the
  strip's middle slot is not written with the placeholder. The
  cross-fade begins the moment the incoming art resolves, either
  through the external preload's handoff or through the carousel's
  own decode. A short safety-net timeout covers the case where
  neither ever resolves: the cross-fade then begins with the
  placeholder, which is the honest end state for a path whose art
  cannot be read.

  A broadcast arriving while the reveal is pending is adopted as
  usual, and if its current path differs from the one being waited
  on, the wait is retargeted to the newest path. The cross-fade
  still begins only when the newest pending path's art resolves.

  The cross-fade itself is unchanged. It uses the same outgoing and
  incoming fractions, the same curves, the same duration, and the
  same small incoming scale-up it has always used. Only the moment
  it begins has moved.

- **The cold-start overlay cannot be left up by a load that never
  completes** — The "Just a moment..." toast and its dim spinner are
  owned by the `LIBRARY_LOAD` operation on the `ToastManager` stack.
  The operation is shown from `onResume` and from the library load's
  entry point, and is hidden by the load's completion callback, the
  load's error callback, or the permission request's denial branch.
  A heartbeat runnable re-posts itself every 500 ms while the
  operation is on the stack, so `ToastManager`'s stuck detector
  knows the operation is alive.

  There was one path with no exit. If the service bind were accepted
  but `onServiceConnected` never fired — the service process
  crashing in `onCreate` before returning its binder, an OOM kill
  between the bind and the callback — the load would never run, its
  callbacks would never fire, and the heartbeat would re-post itself
  forever. The stuck detector watched the heartbeat and saw a live
  operation; it watched the overlay and saw a valid holder. Neither
  of its two stuck conditions could fire.

  The heartbeat is now bounded. After 60 seconds have passed since
  the first heartbeat, the runnable stops and calls the same abandon
  path a load error uses: the overlay and the spinner are hidden
  under the `LIBRARY_LOAD` operation, and a short toast tells the
  user the library could not be loaded. The operation is removed
  from the stack, so the next cold-start attempt starts clean.

  The bind's synchronous return value is checked as well. A refused
  bind is reported through the same abandon path immediately, without
  waiting for the deadline. The incoming-file hand-off is dispatched
  only when the bind was accepted: a refused bind means the service
  cannot be reached, so parking a path would only leave it orphaned.

## Under the Hood

- `FolderManager.performPlaylistUpdate()`'s completion callback no
  longer clears the loose-songs table and no longer calls
  `SongDatabase.deleteSongsNotInSet` with the selected-folder set.
  The loose-songs table is cleared once, in `saveChanges`, before
  the update is dispatched; the songs table is reconciled once, by
  `PlaylistManager.performScan`, against the song paths the scan
  found under the kept folders.

- `AlbumArtCarousel.SwipeState` gained `REVEAL_PENDING`. The state
  is entered when a reveal is deferred because the incoming path's
  art has not yet resolved. While in this state the outgoing art
  stays on screen; no overlay is created and `render()` is not
  called for the incoming window.

- `AlbumArtCarousel` gained `mPendingRevealPath`, holding the path
  the pending reveal is waiting for, and
  `mPendingRevealTimeoutRunnable`, the safety-net timeout that fires
  when the art never resolves. Both are cleared by `cancelSwipe`,
  and `mPendingRevealPath` is retargeted by `setWindow`'s
  `REVEAL_PENDING` branch, which routes the retarget through
  `startReveal(String)` so the new path gets its own wait.

- `AlbumArtCarousel` gained `mDecodedPaths`, a set of paths whose
  decode has resolved — whether or not a bitmap was produced.

- `AlbumArtCarousel` gained `startReveal(String)`,
  `beginPendingReveal(String)`, and `cancelPendingRevealTimeout()`.
  `adoptInIdle`'s reveal branch calls `startReveal` instead of
  `startFade` directly. `scheduleIfMissing`'s decode callback and
  `onArtReady` both resolve a pending reveal whose pending path
  matches the one they resolved.

- `AlbumArtCarousel.REVEAL_OUTGOING_FRACTION` is `0.45f`,
  `REVEAL_OVERLAP_FRACTION` is `0.15f`,
  `REVEAL_INCOMING_START_SCALE` is `0.96f`, and
  `REVEAL_DURATION_MS` is `500L`, matching `COMMIT_DURATION_MS`.
  The reveal runs the same shape it has always run; only the moment
  it begins has moved.

- `AlbumArtCarousel.REVEAL_ART_TIMEOUT_MS` is the new safety-net
  timeout for a pending reveal, in milliseconds.

- `MusicPlayer.ensureLibraryLoadHeartbeat()` bounds the heartbeat
  with `LIBRARY_LOAD_HEARTBEAT_DEADLINE_MS`. The runnable stops when
  the deadline is exceeded and calls
  `abandonLibraryLoadOverlay(String)`, which hides the persistent
  overlay under `OPERATION_LIBRARY_LOAD`, hides the
  `REASON_COLD_START` spinner, and shows a failure toast.

- `MusicPlayer.proceedWithInitialization()` checks the return value
  of `bindService`. A refused bind calls the same
  `abandonLibraryLoadOverlay(String)` path with a bind-specific
  message, and the incoming-file hand-off is skipped.

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