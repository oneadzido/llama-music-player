# Llama Music Player v1.4.0

**Release date:** Nov 6, 2026

---

### Overview

Favorites and playback scope land in this release. A song can now be
marked as a favorite from the player's transport row, from a
double-tap on the album art, or from a heart on any playlist row. The
player can be restricted to favorites, to loose tracks, or to a
chosen set of folders, genres, artists, or albums. A playback-scope
dialog shows a live count of the songs the current selection would
yield.

The favourites feature is protected by a single structural change in
the database write path. `SongDatabase.setFavorite(String, boolean)`
is the only writer of the new `favorite` column on an existing row;
every other metadata write — the scanner, the genre loader, the
metadata editor, the playlist rebuild — goes through
`updateSongMetadata(Song)`, which deliberately omits that column. A
fresh row is inserted with the flag it was built with; an existing
row is routed through the metadata-only path, so a scan that touches
every song in the library cannot silently clear a user's favourites.
The schema bumps to version 3; both tables gain the column with
`DEFAULT 0`.

The playlist is now a three-tier value in `MusicService`.
`mFullPlaylist` is the immutable reflection of the database.
`mScopedPlaylist` is the same list filtered by the active
`PlaybackScope`. `mCurrentPlaylist` is the active queue — the scoped
playlist plus the songs at `mCurrentIndex` and `mTargetIndex`, the
grace period that keeps the currently playing song and its pending
target in the queue even when a scope change no longer includes
them. `rebuildPlaylists()` is the single mutator of the second and
third tiers.

Two further listening controls join the album art in this release. A
centre vertical swipe on the album art resumes a paused song on an
upward flick and pauses a playing one on a downward flick. The
gesture requires twice the touch slop of travel to distinguish it
from digitizer wobble on a resting finger, and it fires on release,
not on the first significant move, so a single drag toggles at most
once. The exit button on the main player screen closes the app on a
deliberate two-tap gesture: the first tap turns the button red and
shows a confirmation toast, the second tap within one second exits.
If the second tap does not arrive, the button reverts to its normal
theme colour so an accidental tap does not leave the exit state
active.

The playlist's edge fast-scroll is now a shared helper,
`EdgeScrollHelper`, installed with one call on any `RecyclerView`.
The playlist and the playback-scope dialog both install it, so the
edge width, engagement threshold, and scroll rate are the same on
both surfaces. A change to any of the three takes effect on both at
once.

---

### Added

- **Favorites** — A per-song favourite flag the user can toggle
  from three places. The transport row carries a sixth button
  between the repeat button and the metadata row; a double-tap on
  the album art opens the same commit; every playlist row carries
  a heart at its trailing edge. In the playlist, a tap on a row's
  heart flips the in-memory flag immediately, redraws the row, and
  forwards the change to a `FavoriteListener` on the owning
  activity. The activity batches the changes in a
  `PendingFavorites` map and flushes them to
  `MusicService.commitFavorites(Map)` in `onPause()`, so a tap
  that is reversed before the activity pauses produces no database
  write at all and the last value staged for each path wins. The
  transport heart and the album-art double-tap commit immediately.
  The heart's tint follows the theme colour when the song is
  favourited and a muted grey when it is not. A
  `FAVORITE_CHANGED` broadcast reaches consumers that render
  hearts without waiting for the next `UPDATE_PLAYER` turn.

- **Playback scope with seven kinds** — A new `PlaybackScope`
  value object with a `Kind` enum — `ALL`, `FAVORITES`,
  `LOOSE_TRACKS`, `FOLDERS`, `GENRES`, `ARTISTS`, `ALBUMS` — and
  four filter sets. `matches(Song, Set)` is the only read; it
  consults `CharacterMapper.isUnknownX(String)` so a song whose
  field is the current locale's placeholder is never matched
  against a stored value. Serializes to and deserializes from
  JSON through the framework's `org.json`; a malformed string
  deserializes to `ALL`.

- **Playback scope dialog** — Reached from a new SCOPE button in
  the folder manager's header, styled to match the playlist's
  SORT button. The dialog presents the seven modes as a single
  flat list and expands a mode's filter list below its row when
  that mode is selected; re-tapping the same mode collapses it
  without changing the selection. Unchecking the last filter
  value in a category returns playback to **All**. The header
  shows a live count of the songs the current selection would
  yield, computed through the same matcher the service uses to
  build the playlist, so the number the dialog shows and the
  queue the service plays cannot disagree. Applying a scope
  broadcasts the serialized scope to the service through
  `APPLY_PLAYBACK_SCOPE`; the folder manager does not bind
  `MusicService` directly. The dialog shares the playlist's edge
  fast-scroll through `EdgeScrollHelper`.

- **Three-tier playlist** — `MusicService` carries
  `mFullPlaylist`, `mScopedPlaylist`, and `mCurrentPlaylist`.
  `rebuildPlaylists()` is the single mutator of the second and
  third tiers; it runs on the main thread and is called from
  `setPlaylist`, `setPlaybackScope`, `commitFavorites`, and at
  the tail of `onPlaybackStarted`. The currently playing song and
  the pending target survive a rebuild through the grace period,
  so unfavoriting the playing song in the Favorites scope does
  not silence it and a scope change mid-transition does not
  interrupt the transition.

- **Exit button** — The main player screen's exit button closes
  the app on a deliberate two-tap gesture. The first tap turns
  the button red and shows a confirmation toast; the second tap
  within one second exits. If the second tap does not arrive, the
  button reverts to the current theme colour so the exit state
  does not linger after an accidental tap on a button the eye
  already reads as a close control.

- **Centre swipe to play or pause** — A vertical swipe on the
  album-art container that starts at the centre becomes a
  playback gesture. Upward resumes a paused song; downward pauses
  a playing one. The gesture requires twice the touch slop of
  travel to distinguish it from digitizer wobble on a resting
  finger, and it fires on release, not on the first significant
  move, so a single drag toggles at most once. The gesture is one
  of four the album-art container classifies on the first
  significant move: a horizontal swipe becomes a carousel swipe,
  a vertical swipe that starts in the left or right edge zone
  becomes a volume change, a vertical swipe that starts at the
  centre becomes the playback gesture, and a touch that never
  leaves the slop radius stays in the resting state from which
  the double-tap favourite is detected. The four classifiers
  share one state enum so a gesture cannot fall into two modes at
  once.

- **Shared edge fast-scroll** — The playlist's edge fast-scroll
  is now a standalone helper, `EdgeScrollHelper`, installed with
  one call on any `RecyclerView`. The playlist and the
  playback-scope dialog both install it, so the edge width,
  engagement threshold, and scroll rate are the same on both
  surfaces.

- **Empty-state message distinguishes scope from library** — A
  library with no songs shows the "import folders" message; a
  library with songs filtered to zero by an active scope shows
  "No songs in the current playback scope."

---

### Under the Hood

- `PlaybackScope` — New class. An immutable value with a `Kind`
  enum and four `Set<String>` fields (`paths`, `genres`,
  `artists`, `albums`). `matches(Song, Set<String>)` consults the
  set the kind names; a null song, a null path, and a field
  carrying the current locale's placeholder all return false.
  `serialize()` produces a JSON object through the framework's
  `org.json`; `deserialize(String)` returns `ALL` for any input
  that does not parse. `ALL` is a static instance.

- `PlaybackScopeDialog` — New class. Top bar with CANCEL, title,
  and APPLY; a single RecyclerView carrying both the seven mode
  rows and the filter rows that belong to the currently expanded
  mode. Mode rows render a custom-drawn radio indicator; filter
  rows render a custom-drawn checkbox indicator; both are themed
  through `ThemeHelper.updateRadio` and
  `ThemeHelper.updateCheckbox`. Folder rows come from the
  caller-supplied `List<FolderAdapter.FolderItem>`; genres,
  artists, and albums are enumerated from
  `SongDatabase.getAllSongs()` on the dialog's executor. The
  enumeration filters out `CharacterMapper.isUnknownX(String)`
  values. Applying a scope whose kind is a filter kind with an
  empty set returns `ALL` rather than a scope that matches
  nothing. The executor is shut down in `dismiss()`. APPLY is
  disabled for the whole window between the tap and the
  dismissal; a preference change during that window cancels the
  queued commit and re-enables the button.

- `Song.isFavorite()` / `Song.setFavorite(boolean)` — New
  accessors. The `mFavorite` field defaults `false`, is set by
  the row reader when the schema carries the column, and is
  preserved across the `serialVersionUID` bump to 2.

- `SongDatabase.DATABASE_VERSION` — Bumped to 3. `createTable`
  and `onUpgrade` add the `favorite INTEGER DEFAULT 0` column to
  both tables.

- `SongDatabase.buildContentValues(Song)` — Splits into a full
  builder and a metadata-only builder. The full builder is used
  by the fresh-row branch of the insert path and includes the
  favorite flag. The metadata-only builder is used by the update
  branch and by `updateSongMetadata(Song)`, and deliberately
  excludes the flag.

- `SongDatabase.insertSong(Song)` — Routes an existing row
  through `updateSongMetadataInternal`; a fresh row goes through
  `insertSongInternal` with the full content values. The batch
  insert (`insertSongsBatchInternal`) does the same for each song
  inside its transaction.

- `SongDatabase.updateSongMetadata(Song)` — New public method.
  Updates every column except `favorite`.

- `SongDatabase.setFavorite(String, boolean)` — New public
  method. Writes the `favorite` column on both the songs and
  loose-songs tables.

- `SongDatabase.RowReader` — Resolves the `favorite` column
  alongside the others; when present, sets the flag on the
  returned song. When absent (a cursor from an older schema), the
  song defaults to not favorited.

- `MusicService.mFullPlaylist` — New field. The immutable
  database reflection, assigned by `setPlaylist`.

- `MusicService.mScopedPlaylist` — New field. `mFullPlaylist`
  filtered by `mPlaybackScope`. Recomputed only by
  `rebuildPlaylists()`.

- `MusicService.mPlaybackScope` — New field. The active scope,
  defaulting to `PlaybackScope.ALL`.

- `MusicService.mLooseTrackPaths` — New field. The set of paths
  in the loose-songs table, refreshed by `rebuildPlaylists()`.

- `MusicService.rebuildPlaylists()` — New private method. The
  single mutator for `mScopedPlaylist` and `mCurrentPlaylist`.
  Captures the paths at `mCurrentIndex` and `mTargetIndex` before
  rebuilding; adds the songs back into the new list if the scope
  excluded them; re-anchors both indices to the captured paths;
  clears the shuffle peek memos, the shuffle history, and the
  cached window.

- `MusicService.setPlaybackScope(PlaybackScope)` — New public
  method. Stores the scope, persists it under
  `playback_scope_json`, rebuilds the playlists, and broadcasts.

- `MusicService.getPlaybackScope()` — New public method. Returns
  the active scope.

- `MusicService.getScopedPlaylist()` /
  `MusicService.getScopedPlaylistSize()` — New public methods.
  Return a copy of `mScopedPlaylist` and its size.

- `MusicService.commitFavorites(Map<String, Boolean>)` — New
  public method. Applies the map to all three lists, persists
  each path through `SongDatabase.setFavorite`, rebuilds the
  playlists, broadcasts `UPDATE_PLAYER`, and broadcasts
  `FAVORITE_CHANGED`. A per-path database failure is logged and
  does not abort the batch.

- `MusicService.toggleFavorite()` — New public method.
  Convenience wrapper that commits a single-path change for the
  current song.

- `MusicService.broadcastUpdate()` — Carries three new extras:
  `is_favorite` (the current song's flag), `scope_kind` (the
  active kind's name), and `scope_count` (the scoped playlist
  size). The activity reads `scope_count` to present the no-song
  state when the active scope yields no songs without waiting for
  a song to appear.

- `MusicService` — New `ACTION_APPLY_PLAYBACK_SCOPE` broadcast
  constant, `EXTRA_SCOPE_JSON` extra key, `ACTION_FAVORITE_CHANGED`
  broadcast constant, and a receiver that applies a serialized
  scope on the main thread. The receiver is registered in
  `onCreate` and unregistered in `onDestroy`.

- `MusicService.loadPlaybackScope()` /
  `MusicService.savePlaybackScope()` — New private methods.
  Deserialize and write `playback_scope_json`.

- `PlaylistActivity.PendingFavorites` — New static inner class. A
  `HashMap<String, Boolean>` with `stage(path, flag)`,
  `isEmpty()`, `snapshot()`, and `clear()`. The playlist row's
  `FavoriteListener` stages; `onPause` flushes and clears.

- `PlaylistActivity.setupAdapter()` — Installs a
  `FavoriteListener` on the adapter that stages heart taps in
  `PendingFavorites`.

- `PlaylistActivity.onPause()` — Flushes `PendingFavorites` to
  `MusicService.commitFavorites` before clearing the map and
  unregistering the theme receiver.

- `PlaylistActivity.showEmptyView(int)` — Takes a string
  resource identifier. `refreshPlaylistFromService` passes
  `playlist_empty_scope` when the active scope is not `ALL`,
  `playlist_empty_state` otherwise;
  `registerLibraryResetReceiver` passes `playlist_empty_state`
  unconditionally, because a reset is always a library-empty
  state.

- `PlaylistActivity.setupEdgeFastScroll()` — Now calls
  `EdgeScrollHelper.attach(mPlaylistRecyclerView)`. The
  playlist's own inlined fast-scroll implementation is deleted;
  the helper is the single implementation on both surfaces.

- `PlaylistActivity.applyThemeToUI()` — Rebinds every visible row
  on a theme change through
  `notifyItemRangeChanged(0, getItemCount())`, so every visible
  heart picks up the new tint.

- `PlaylistAdapter.FavoriteListener` — New interface. Called
  after the row's `Song` flag is flipped and the row is rebound.

- `PlaylistAdapter.ViewHolder` — Resolves `R.id.favoriteIcon`
  alongside the title and subtitle. Binds the heart's glyph,
  tint, content description, and click handler in
  `onBindViewHolder`.

- `MusicPlayer.mFavoriteButton` — New field. Resolved in
  `initializeViews` after `mMetadataButton`; typeface, glyph,
  background, and animation set alongside the other control
  buttons. Its click goes through `MusicService.toggleFavorite`
  with the standard 500ms debounce.

- `MusicPlayer.mExitButton` — New field. Resolved in
  `initializeViews`. Click handler tracks `mExitPending` and
  schedules `mResetExitRunnable` to clear the pending state after
  the toast's 1000 ms duration; a second tap while
  `mExitPending` calls `finishAffinity()`.

- `MusicPlayer.refreshTransportControls()` — Reads the current
  song's favorite flag alongside the playing, repeat, and
  shuffle states and writes the heart's active tint through the
  same `updateControlButton` call the other four buttons use.

- `MusicPlayer.handleGestureDown` / `handleGestureMove` /
  `handleGestureUp` — The gesture mode enum renames `UNDECIDED`
  to `TOUCHED`, which is now both the pre-classification state
  and the state from which a double-tap is detected. On
  `ACTION_UP` in `TOUCHED`, the handler compares the tap time
  against the last recorded tap; two taps inside
  `ViewConfiguration.getDoubleTapTimeout()` call
  `toggleCurrentFavorite` and reset the timestamp.

- `MusicPlayer.toggleCurrentFavorite()` — New private method.
  Reads the current song's flag, calls
  `MusicService.toggleFavorite`, and shows the appropriate
  favorite toast. The read and the toggle run under
  `mServiceLock` so a concurrent transport tap cannot
  interleave.

- `MusicPlayer.handlePlaybackGesture(float)` — New private
  method. Fires from `handleGestureUp` when the state is
  `PLAYBACK` and the release is an `ACTION_UP`. Requires twice
  the touch slop; calls `mMusicService.resume()` on an upward
  swipe against a paused or ready song, and
  `mMusicService.togglePlayPause()` on a downward swipe against
  a playing song. No-op when the song is already in the target
  state.

- `MusicPlayer.applyNoSongState()` — Resets the favorite button
  to the inactive tint when the empty state is applied.

- `FolderManager.mScopeButton` — New field. Resolved in
  `initializeViews`; its click handler calls
  `openPlaybackScopeDialog`. Its enabled state is written by
  `updateFolderButtons` from the same `isUpdating` gate the
  IMPORT button uses.

- `FolderManager.openPlaybackScopeDialog()` — New private
  method. Reads the current scope from preferences, constructs a
  `PlaybackScopeDialog`, and, on apply, broadcasts
  `APPLY_PLAYBACK_SCOPE` with the serialized scope.

- `FolderManager.updateFolderButtons()` — Includes the scope
  button in the enable/disable pass, gated on `!isUpdating()`.

- `EdgeScrollHelper` — New class. Installs an
  `OnItemTouchListener` on any `RecyclerView`. A vertical drag
  that starts inside the list's leftmost or rightmost 32dp
  scrolls proportionally: a full viewport-height of finger
  travel maps to five percent of the total scrollable range,
  anchored at the position the drag began. The gesture engages
  only when the content is at least three times taller than the
  viewport; on a shorter list the edge zones defer to the
  RecyclerView's own scrolling. A tap is never consumed. The
  playlist and the playback-scope dialog both install it, so a
  change to the edge width, the engagement threshold, or the
  scroll rate takes effect on both surfaces at once.

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
| Favorites | Double-tap the album art, tap the heart beside the transport controls, or tap a playlist row's heart |
| Playback scope | Tap MANAGER, then SCOPE in the folder manager header |
| Exit button | Tap the exit button on the main screen twice within one second |
| Equalizer | Tap EQUALIZER button on main screen |
| Metadata Editor | Tap METADATA button on main screen |
| Folder Manager | Tap MANAGER button on main screen |
| Playlist | Tap PLAYLIST button on main screen |
| Themes | Tap LLAMA button on main screen, then THEMES |
| Language | Tap LLAMA button on main screen, then the language button in the version header |
| Volume gesture | Swipe up or down along the album-art edge |
| Centre swipe to play or pause | Swipe up on the album art to resume, down to pause |
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