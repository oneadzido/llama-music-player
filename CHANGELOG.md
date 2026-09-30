# Changelog

All notable changes to Llama Music Player are documented in this file.

The sections at the bottom of this file — System Requirements, Download,
Installation, First Time Setup, How to Use, Credits, and License —
describe the current release and are not repeated per entry. A release
that changed one of those facts says so inside its own entry.

## [1.2.0] - Sep 30, 2026

### Overview
Two new interaction features, a split-screen layout that reshapes the
whole app, a redesigned loading indicator, a process-wide overlay
architecture, and a three-slot album-art carousel that renders the
playback window the service publishes.

### What's New
- **Volume-aware playback** — Reaching volume 0 pauses playback.
  Raising the volume above 0 resumes it. The rule is symmetric and
  independent of what caused the pause: the user's act of raising
  the volume is itself a signal that they want to hear the music.
- **Album-art volume gesture** — Swipe up or down along the left or
  right edge of the album art to change the volume. A short toast
  shows the current level as a percentage. When the gesture reaches
  the device maximum or zero and the finger continues in the same
  direction, the toast echoes at a fixed rhythm so the gesture still
  reads as being tracked even though the device volume cannot move
  further.
- **Playlist edge fast-scroll** — A vertical drag that starts inside
  the leftmost or rightmost 32dp of the playlist scrolls the list
  proportionally: a full viewport-height of finger travel moves
  through five percent of the total scrollable range, anchored at
  the position the drag began. On a library large enough for the
  proportional mapping to be useful, the end of the list is reachable
  in a bounded number of gestures regardless of how many songs it
  holds.
- **Sort progress feedback** — Cycling the playlist's sort mode shows
  the same dim-and-spinner overlay the app uses for longer
  operations. The overlay appears the moment the sort starts and
  disappears when the sort finishes, with a short floor so it does
  not flash on a small library.
- **Split-screen layout** — Every activity declares the same
  `configChanges` set and adjusts its own content when the window
  resizes instead of being recreated. In the player, the album art
  resizes with the window: at full height it is shown at 300dp; in a
  tall split window it shrinks to 180dp and the volume gesture
  remains available; in a short split window the art hides so the
  title, artist, progress bar, timer, and controls take the space.
  In the equalizer, the band controls and the effect controls share
  a single scroll container; in split mode the band wrapper drops
  its weight and takes its natural height, so the user scrolls
  through both sections as one column. The folder manager hides the
  folder list and shows a centred placeholder in its slot. The
  metadata editor and the playlist let their existing scroll
  containers absorb the overflow.
- **Redesigned loading indicator** — A rotating ring with a bright
  arc travelling around a dim track, in place of the previous static
  circle. The arc's position is legible on every frame, so the
  indicator reads as moving regardless of how long the underlying
  work takes.

### Album-art carousel

The album art is a three-slot strip: the previous song's art on the
left, the current song's art in the middle, the next song's art on
the right. Swipe horizontally across the art and the strip follows
your finger. Release past the midpoint and the strip commits to the
neighbouring song; release before it and the strip springs back.

The carousel renders a window the service publishes — the previous
neighbour's path, the current song's path, the next neighbour's path,
computed by `MusicService.getWindowSnapshot()`. The window is the same
triple the service's own Next and Previous requests commit from, so
the carousel and the service cannot disagree about which song will
play next, which song is the previous, or which song is current. A
window is a value; adopting it is a single method, `setWindow`, and
every path through the service — a transport button, a headset key, a
notification action, a Bluetooth resume, a shuffle re-pick, a sort
reorder, a library reset — publishes a window.

Pressing Next or Previous animates the same strip. The slide and the
audio transition run in parallel: the button dispatches the service
request immediately, and the carousel slides while the service
prepares the next song. When the service's window arrives, the strip
settles on the incoming art. The visual result is identical to a
committed swipe.

Under shuffle the service's next is the shuffle-chosen index, not a
positional neighbour; the carousel renders whatever path the service
publishes, and the shuffle pick is memoised so the song the carousel
displayed in its next slot is the song the service plays. Rapid
skip-forward and skip-back are handled by the path's own identity: the
carousel installs a decoded bitmap only into a slot whose path matches
the window, and a decode that resolves after the window has moved on
is placed in the cache and dropped.

### Playlist edge fast-scroll

The playlist list owns a fast-scroll zone on each of its two vertical
edges. A vertical drag that starts inside the leftmost or rightmost
32dp of the list scrolls the list proportionally: a full
viewport-height of finger travel moves through five percent of the
total scrollable range, anchored at the position the drag began. The
mapping is smooth and reversible — the same finger position always
lands on the same offset — so the user can read rows while dragging
past them and reverse direction without the list jumping.

Fast scroll only engages when the content is at least three times
taller than the viewport. On a shorter list the proportional mapping
would be too coarse for a user who is reading rows, and the edge
zones defer to the RecyclerView's own scrolling. Below that
threshold, the two edge zones behave exactly like the rest of the
list.

A tap is never consumed by the edge zones. The gesture classifier
takes over the sequence only after the finger has moved past the
touch slop, so tapping a song that happens to sit near an edge still
plays it. Once the classifier has taken over, it cancels any scroll
the RecyclerView had already begun in its slop window and drives the
rest of the gesture itself, so the two scroll systems never fight
over the same drag.

The gesture is independent of the album-art carousel and of the
volume gesture. It lives on the playlist list; the carousel's swipe
and the album-art volume gesture live on the album-art container in
the player screen. No screen hosts both.

### Media carousel

The notification and the system media controls follow every playback
transition, including rapid skip-forward and skip-back from the
carousel itself. The metadata is delivered to the notification and
the media session as it arrives, with the themed placeholder as the
large icon while a song's album art is still decoding, and with the
real bitmap once the decode resolves. The playback state is written
on every awareness tick, so a state change the carousel missed on
one tick is retried on the next.

The notification, the media session, the player UI, and the album-art
carousel all read their display identity from one rule on the
service: during a transition the display song is the target;
otherwise it is the current song. One expression of the rule, four
consumers, no drift.

### Playback restoration

The last song and its playback position restore reliably on every
cold start, including after a force-kill or an OS-initiated process
teardown. The position is checkpointed on a fixed interval while
music is playing, alongside the existing saves at play, pause, and
seek. A force-kill loses at most five seconds of listening position
instead of the whole song.

### Folder import

Importing a folder after a library wipe prepares the first song
paused. The service has no prior playback signal to honour on a cold
start, so it waits for you to press play rather than starting
playback on its own. Restoring a session after a restart is
unaffected: if you were playing when the process went away,
playback resumes at the saved position.

### Loading overlay

Cold start, the metadata editor's reads, the folder-update cycle, and
a user-initiated sort of the playlist all show the same
dim-and-spinner overlay. The overlay is process-wide and
reason-keyed: the spinner and the dim background belong to the
process, not to any single activity, and the overlay follows the
user across navigation. When you back-swipe out of the folder
manager while a scan is running, the overlay is still there when you
land in the player. The playlist rebuild now uses the in-app overlay
instead of the previous background notification, so the transition
from folder scan to genre loading stays inside the app's own surface.
The spinner's rotation is bound to its attach state, so the animation
runs exactly while the view is on screen.

### Under the Hood

- `PlaybackWindow` — an immutable value type holding the three paths
  the carousel renders: the previous neighbour, the current song,
  the next neighbour. Equality compares the three paths, so
  republishing the same window is a no-op for every consumer. The
  service is the sole producer; the carousel is the sole consumer.
- `MusicService` is the sole authority for the window.
  `getWindowSnapshot()` computes it from two pure functions,
  `peekNextIndex(int)` and `peekPreviousIndex(int)`, that are the
  same primitives the service's own `playNextSong()` and
  `playPreviousSong()` commit from. The peek functions do not mutate
  the shuffle history; they memoise the answer the next commit will
  consume, so the index the window advertised is the index the
  service plays. The memo is cleared on every operation that changes
  what an index means: a playlist replacement, a sort reorder, a
  shuffle toggle, a repeat-mode change, a transition to idle, and a
  commit. The shuffle history is cleared on the same operations
  except a repeat-mode change and a commit; on a commit it is
  appended to for a next, or popped from for a previous.
- `MusicService.getDisplaySong()` is the single expression of "which
  song is the app showing right now". During PREPARING it is the
  target; every other state uses the current index. It feeds
  `snapshotForNotification`, `getWindowSnapshot`, `broadcastUpdate`,
  and `prefetchNotificationArt`. There is no second expression of
  the rule, so there is nothing for one consumer to drift against
  another. The notification and the media session carousel
  therefore advance with the user's tap on a next or previous
  request, in step with the app carousel, rather than waiting for
  the transition to complete.
- `AlbumArtCarousel` — the three-slot album-art strip. It holds no
  playlist, no index, and no bitmap as primary state. Its entire
  state is the last window it was given, the current commit state,
  the confirmed window during a running animation, and the shared
  placeholder bitmap. It renders the three paths the service names
  and consults the shared `LruCache` for each path's art. A single
  public mutator, `setWindow`, adopts every window, whether the
  window arrived as the result of a swipe, a button press, a
  headset key, or a Bluetooth resume.
- `AlbumArtCarousel.animateToNext()` and
  `AlbumArtCarousel.animateToPrevious()` — the button-driven
  commit. The caller dispatches the service request immediately and
  the carousel slides in parallel. The carousel does not dispatch
  the swipe commit listener for a button-driven commit; the caller
  that started the animation is responsible for dispatching the
  service request.
- `AlbumArtCarousel` has four commit states. `IDLE` is the resting
  state. `DRAGGING` is a swipe in progress. `COMMITTING` is a
  strip animating to a neighbour position. `AWAITING_SNAPSHOT` is a
  commit dispatched, waiting for the service's window. The expected
  current path is set before the animation starts, so a window that
  arrives during the animation can be matched immediately: a
  matching window is stored and adopted when the animation
  finishes, and the strip settles at rest without a jump. A
  non-matching window is a refusal, and the strip springs back. A
  window that arrives while the user is dragging is adopted
  directly; the drag is abandoned.
- `PlaylistActivity.setupEdgeFastScroll()` — installs the
  fast-scroll interceptor on the playlist `RecyclerView` through an
  `OnItemTouchListener`. The interceptor records the anchor at
  `ACTION_DOWN`, engages on the first `ACTION_MOVE` that exceeds the
  touch slop inside an edge zone and on a list long enough for the
  proportional mapping, then cancels the RecyclerView's own scroll
  and drives the rest of the gesture through `scrollBy`. A tap is
  never consumed: the interceptor returns false from
  `onInterceptTouchEvent` until the slop is crossed.
- `PlaylistActivity.cycleSortMode()` — shows
  `OverlayManager.REASON_PLAYLIST_SORT` before dispatching the sort
  to a background thread, and schedules the hide from the sort's
  main-thread completion callback. The hide is scheduled with
  `SORT_OVERLAY_MIN_DURATION_MS` minus the sort's own elapsed time,
  so the overlay does not flash on a small library. A subsequent
  cycle cancels the previous hide before scheduling its own, so
  rapid taps do not stack. The reason is also released in
  `onDestroy` if a sort was still in flight when the activity went
  away.
- The split-screen behaviour is per-activity, not shared. Each
  activity applies its own layout rule in `onConfigurationChanged`.
  `MusicPlayer.applyAlbumArtSize` reads `Configuration.screenHeightDp`
  and sets the album art's size and visibility.
  `EqualizerActivity.applyEqualizerSplitLayout` reads the same value
  and toggles the band wrapper's layout weight: 1 in the full layout
  so the band controls take the space the effect controls leave, 0
  in split mode so the wrapper takes its natural height. The band
  controls and the effect controls live inside one scroll container
  either way, so in split mode the two sections scroll together and
  in the full layout the whole column fits the window.
  `FolderManager.applyFolderManagerSplitLayout` hides the list and
  shows a centred placeholder in its slot. `MetadataEditor` and
  `PlaylistActivity` let their existing scroll containers absorb the
  overflow. Every activity declares the same `configChanges` set, so
  a resize runs `onConfigurationChanged` instead of recreating the
  activity.
- `OverlayManager` — the process-wide, reason-keyed dim-and-spinner
  overlay. A singleton that registers itself as an
  `Application.ActivityLifecycleCallbacks` listener and reattaches
  the overlay to the current activity on every resume. The overlay
  is visible while at least one reason token is held, so the
  cold-start library load, the metadata editor's reads, the
  folder-update cycle, and a user-initiated sort can all hold the
  overlay without interfering with each other. The spinner is a
  private view with a lifecycle-bound rotation animator, started on
  attach and cancelled on detach. A periodic health check releases
  any reason held longer than the stuck timeout.
- `MusicService` monitors the music stream through a receiver for
  `AudioManager.VOLUME_CHANGED_ACTION`, filtered to the music
  stream. A drop to zero pauses playback when the state is PLAYING,
  or defers a pause when the state is PREPARING; a raise from zero
  resumes playback when the state is PAUSED, or clears the deferred
  pause when the state is PREPARING. The rules are symmetric and
  independent of what caused the pause.
- `MusicService` checkpoints the persisted position on a fixed
  interval inside the existing health monitor, and on
  `onTrimMemory`. The state-change saves remain in place for pause,
  seek, and stop.
- `MusicService.setPlaylist` prepares the anchor paused on a cold
  start that has no prior playback signal to honour, which is the
  case after a library wipe or a fresh install. When the persisted
  state records that playback was active before the process went
  away, the anchor is prepared to resume at the saved position
  instead. The rule applies whether the anchor came from a
  persisted last-song path or from the fallback to index zero on an
  empty playlist history.
- `NotificationController.applySnapshot` commits a song's identity
  as it arrives, with the themed placeholder as the large icon when
  art has not yet resolved and the real bitmap when it has. The
  snapshot the controller commits is the one
  `MusicService.snapshotForNotification` produced, so the
  notification, the media session, the player UI, and the app
  carousel all read the same display song.
- `MusicPlayer.onConfigurationChanged` reapplies the album-art size
  and visibility. The decision is a pure function of
  `Configuration.screenHeightDp`.
- `MusicPlayer.proceedWithInitialization` is linear: preload the
  previous song for display, then start and bind the service. The
  activity's preload paints the title, artist, and art while the
  playlist loads, and does not decide whether anything is restored.
- `MusicPlayer`'s album-art container owns every gesture that
  operates on the album art. A single touch listener classifies each
  gesture on the first significant move: a horizontal swipe becomes
  a carousel swipe, a vertical swipe that starts inside the left or
  right 20dp of the container becomes a volume change, and a
  vertical swipe that starts at the centre is cancelled so it
  reaches whatever sits behind the album art.
- `MusicPlayer.playNextSong()` and `MusicPlayer.playPreviousSong()`
  drive the carousel's slide before dispatching the service request.
  When the carousel cannot animate — a commit is already in flight,
  the viewport is not laid out, or the window advertises no
  neighbour in that direction — the call is a no-op and the service
  request still proceeds; the resulting window is adopted without
  animation.

---

## [1.1.9] - Sep 28, 2026

### Overview
A small maintenance release. One user-visible bug fix, two message and
timing adjustments, and the supporting changes underneath them.

### Fixed
- **Media carousel settling on a previous song under rapid switching** —
  Two independent causes, both fixed.

  The awareness poll recorded a snapshot as rendered whether it had been
  rendered or held. A held snapshot — a new song whose album art is still
  decoding — was therefore marked as the current render, and a subsequent
  snapshot that compared equal to it was skipped. The poll now records a
  snapshot only when it commits, so a held render is re-evaluated on
  every tick.

  Separately, the session's playback state was written only when a
  metadata commit happened, and on a rapid switch the state value is the
  same before and after (`PLAYING` → `PLAYING`). The carousel therefore
  received a metadata-only delivery, which does not reliably refresh it.
  The session's playback state is now written on every tick from the
  state machine's own state — including the `BUFFERING` state every song
  switch passes through — independent of the metadata commit. A state
  change is delivered on every transition, so the carousel converges on
  the current song no matter how rapidly the user taps.

### Changed
- **Playlist sync overlay message** — A playlist scan triggered by a
  storage change now shows `Syncing playlist...` for the duration of the
  run instead of the metadata loader's default `Loading metadata... X/Y`
  text. The new message names the action the user caused and stays in
  place for the whole operation; the counter is not shown because a sync
  that finds nothing to change would sit at `0/N` and vanish. The
  genre-loading operation itself is unchanged: callers that supply no
  message keep the default progress text.
- **Playlist updated confirmation** — The confirmation toast now uses
  the normal duration instead of the short one.

### Under the Hood
- `PlaylistManager.EXTRA_TRIGGER_MESSAGE` — optional overlay message
  carried on the genre-loading trigger broadcast.
- `PlaylistManager.triggerGenreLoading(String)` — takes an optional
  message; the no-argument overload preserves the previous behaviour.
- `NotificationController.ServiceStateSource` gains
  `sessionPlaybackState()` and `updateSessionPlaybackState()`. The
  session's metadata is a projection of the render commit and commits
  once per song with the art held until it resolves. The session's
  playback state is a projection of the state machine and is written on
  every sync tick, independent of the metadata. The two projections have
  independent lifetimes; only the second is what keeps the carousel in
  sync on a rapid switch.
- `NotificationController.syncOnce()` (renamed from `pollOnce`) records
  a snapshot as rendered only when it commits, so a held render is
  re-evaluated on every tick.

---

## [1.1.8] - Sep 27, 2026

### Overview
This release restructures roughly half the non-UI classes. Four subsystems were rewritten end to end, two classes were renamed, and the framework media session classes were migrated to the AndroidX support library. The user-facing behavior is largely unchanged; the code underneath is not.

### What's New
- **Notification ownership inverted** — `NotificationController` replaces `NotificationService`. The controller polls the service through a state-source interface instead of receiving pushed state, and is now the sole writer of the media session metadata.
- **Media session and notification committed together** — one `RenderSnapshot`, one main-thread turn, two projections. The carousel and the notification can no longer show different songs on a rapid switch.
- **Operation stack replaces the single-owner overlay token** — `ToastManager` renders from an ordered stack of operations. An operation below the top is retained but not shown; a cancelled operation is removed as a terminal state, and the overlay cannot freeze.
- **Folder-update cycle overlay ownership made strictly sequential** — folder sync, playlist rebuild, and genre loading each own the persistent overlay for the duration of their own work. One operation per phase, pushed and popped by its owner, no operation ever left on the stack underneath another.
- **Empty library leaves no trace** — the media session is created inactive and the service does not promote to foreground when the playlist is empty. A first launch with no songs shows nothing in the status bar or the media carousel.
- **Library reset added** — a first-class `PlaylistManager.resetLibrary()` operation with its own write barrier, invalidation broadcast, and completion signal.
- **Media session migrated to AndroidX** — `MediaSessionCompat`, `PlaybackStateCompat`, `MediaMetadataCompat`, and `MediaButtonReceiver` replace the framework classes.
- **Bluetooth resume rebuilt** — a dedicated `ACTION_BLUETOOTH_RESUME` service action replaces the generic play broadcast. A2DP disconnect now pauses playback.
- **Result types replace ad-hoc returns** — `MetadataReader`, `MetadataWriter`, and `CharacterMapper` now return typed results with exhaustive status enums.
- **Equalizer settings made authoritative in `SharedPreferences`** — getters read what setters wrote, regardless of any rounding the underlying effect applies.

### Technical Improvements

**Folder-Update Cycle**
- `FolderManager` pushes `FOLDER_SYNC` on the scan's first progress callback and pops it at scan completion on every exit path — success, error, and full-wipe completion
- `MusicPlayer`'s `UPDATE_PLAYLIST_FROM_FOLDERS` receiver pushes a `PLAYLIST_REBUILD` persistent overlay before starting the rebuild on its executor, and pops it once the rebuild has been handed to the service
- `MusicPlayer` broadcasts `PlaylistManager.ACTION_TRIGGER_GENRE_LOADING` directly after the rebuild
- The rebuild overlay is pushed with `ToastManager.getCurrentActivity()`, so it renders in whichever window is visible and follows the user across navigation via `ToastManager`'s existing reattach mechanism
- The `PLAYLIST_REBUILD` overlay is heartbeated by `MusicPlayer` for the duration of the rebuild
- `PlaylistManager` pushes `GENRE_UPDATE` and owns it for the whole genre-load lifetime

**Architecture Overhaul**
- **Renamed and Restructured**
  - `NotificationService` → `NotificationController` (inverted ownership)
  - `PlaylistService` → `PlaylistManager` (rename plus ownership change for library reset and genre loading)
- **Framework → AndroidX Support Library**: All media session types (`MediaSession`, `PlaybackState`, `MediaMetadata`) migrated to the `compat` namespace
- **Result Types Replace Ad-Hoc Returns**: `MetadataReader`, `MetadataWriter`, and `CharacterMapper` now return typed results with exhaustive status enums
- **Data Layer Restructured**: `SongDatabase` gained a `Table` enum, a `RowReader` that caches column indexes, and an asynchronous constructor
- **New Operation: Library Reset**: `PlaylistManager.resetLibrary()` with admission, write barrier, invalidation broadcast, and completion signal
- **Notification Ownership Inverted**: `NotificationController` is the sole writer of the media session metadata, committing it from the same `RenderSnapshot` as the notification
- **The Operation Stack**: `ToastManager` now uses an ordered stack of operation IDs to manage persistent overlays, allowing concurrent operations and preventing stuck overlays
- **Metadata Editor Broadcast Unified**: The editor publishes a single `METADATA_UPDATED` broadcast after a successful write. The previous two-broadcast sequence (`UPDATE_SONG_IN_PLAYLIST` followed by `FORCE_UPDATE_PLAYER`) is gone. Each consumer reacts to the one broadcast according to its own concerns, and the service's receiver pokes the controller's awareness poll so the edit reaches both the notification and the session in one turn.
- **Metadata Cache File Extensions Derived from Source**: `MetadataReader` and `MetadataWriter` now determine the cache file's extension from the source document's display name. Naming every cached file `.mp3` would force jaudiotagger's MP3 reader onto FLAC, OGG, M4A, and MP4 sources.
- **Dialog Top-Bar Geometry Unified**: `PresetDialog`, `ThemeDialog`, and `CreditsDialog` now use the same top-bar geometry. The CANCEL button measures and positions identically across all three.

**Playback and State Management**
- **Single Writer for Media Session**: `NotificationController.applySnapshot` is the sole writer of session metadata and playback state, ensuring they cannot diverge
- **Playback State Machine**: Added `mStartToken` to identify valid start/resume continuations, and `mPauseAfterPrepare` to correctly handle a pause during preparation
- **Bluetooth Resume**: A dedicated `ACTION_BLUETOOTH_RESUME` action is now used for a more reliable resume experience
- **Folder Sync and Genre Loading**: Genre loading now runs entirely on `PlaylistManager`'s executor, with a token-based cancellation system

**Build and Platform**
- Java 11 upgraded to **Java 17**
- `encoding 'UTF-8'` set explicitly
- `androidx.core:core` upgraded to **1.13.1** and `androidx.media:media` upgraded to **1.8.0**
- `viewBinding` explicitly disabled
- **Manifest**: Registered `MediaButtonReceiver`, simplified `MusicService` intent filter, updated `BluetoothReceiver` gating, and added `parentActivityName` to all activities

---

## [1.1.7] - Sep 24, 2026

### Fixed
- **MediaPlayer Crashes During Rapid Transitions**: Resolved an issue where rapid or overlapping next/previous requests could leave the MediaPlayer in an invalid state and terminate the playback service:
  - **Serialized Transitions**: A single-gate state machine now ensures only one transition is in flight at a time
  - **Stale Callback Rejection**: Every player callback carries a generation stamp; callbacks from superseded transitions are ignored
  - **Unified Release**: Every MediaPlayer release now runs on the player thread under a single lock, eliminating use-after-release
- **Unbounded Skip Loop on Unreadable Playlists**: Resolved an issue where a playlist full of unreadable files kept the auto-skip loop running indefinitely:
  - **Bounded Budget**: Auto-skip now gives up after 2.5 seconds of consecutive failures
  - **Preserved State**: The playlist and target survive, so a subsequent play request retries with a fresh budget
  - **User Preemption**: Any explicit play, next, or previous press clears the budget immediately

### Technical Improvements
- **Playback Threading**
  - MusicService now runs every raw MediaPlayer call on a dedicated HandlerThread; the state machine stays on main
  - `beginTransition()` posts release, reset, setDataSource, and prepareAsync as one player-thread runnable
  - `onPrepared`, `onCompletion`, and `OnErrorListener` post their continuations back to main with a captured generation
  - `onDestroy()` posts the release, calls `quitSafely()`, and joins with a 2000 ms bound
- **State Machine**
  - `reconcile()` is the sole gate between "a target exists" and "a transition begins"; nothing else calls `beginTransition()`
  - `mGeneration` increments on every transition; stale callbacks return without touching state
  - The two-index model (`mCurrentIndex` / `mTargetIndex`) is unchanged; only the threading beneath it moved
- **Position Path**
  - `getCurrentPosition()` returns an interpolated estimate from a main-thread anchor
  - `refreshPositionFromPlayer()` posts a real read to the player thread every 500 ms
  - A `mPositionEpoch` counter discards position reads that return after a seek
  - `getDuration()` and `isPlaying()` read cached fields
- **Skip Budget**
  - `skipBudgetExhausted()` records the first consecutive failure time and returns true once `SKIP_LOOP_BUDGET_MS` is reached
  - `abandonSkipLoop()` releases the player, clears transition state, and preserves the playlist and target
  - Every user request method clears `mSkipLoopStartedElapsed` at entry
- **Client-Side Changes**
  - MusicPlayer dropped `mControlsTransitioning`; the service serializes next/previous natively
  - MusicPlayer dropped the `forceClearTransitionState()` workarounds; the new state machine cannot get stuck
  - MusicPlayer dropped `mFileHandler`, an unused handler retained for a code path that no longer exists
  - PlaylistActivity gates UPDATE_PLAYER broadcasts by generation and defers auto-scroll while the display list mutates

---

## [1.1.6] - Sep 23, 2026

### Fixed
- **Playback Stopping After Long Sessions**: Resolved an issue where the player sometimes stopped after several hours of playback:
  - **Transition State Clearing**: Playback state now resets on every exit from a transition
  - **Player Recovery**: Internal audio errors now recover without terminating the service

### Technical Improvements
- **Playback State Management**
  - `resetPlayerSafely()` clears `mIsTransitioning` and `mIsPreparing` before touching the MediaPlayer
  - `releaseMediaPlayer()` clears `mIsPreparing` after the player is released
  - The `OnErrorListener` clears both flags on entry before attempting recovery
  - `prepareAndPlay()` and `prepareOnly()` perform release, create, reset, and setDataSource inside a single `mPlayerLock` block
  - `playNextSong()` and `playPreviousSong()` post `playSong()` to the main-thread handler

---

## [1.1.5] - Sep 2, 2026

### Fixed
- **Memory Leak in MusicPlayer.onDestroy()**: Resolved several memory leak issues in the MusicPlayer activity's cleanup process:
  - **MusicLibrary Executor Shutdown**: The MusicLibrary instance now properly calls `shutdown()` on its internal executor service in `onDestroy()`
  - **Album Art Task Cleanup**: Added proper cancellation of `mCurrentAlbumArtFuture` and `mCurrentMetadataFuture` before activity destruction
  - **Album Art Cache Eviction**: The LRU cache now calls `evictAll()` and is nullified to free bitmap memory
  - **Placeholder Bitmap Cleanup**: Themed placeholder bitmaps are now properly recycled via `cleanupPlaceholderBitmaps()`

### Technical Improvements
- **Memory Management**
  - Added `mMusicLibrary.shutdown()` to release executor resources
  - Cancel pending album art futures (`mCurrentAlbumArtFuture`, `mCurrentMetadataFuture`)
  - Evict and nullify `mAlbumArtCache` to free bitmap memory
  - Call `cleanupPlaceholderBitmaps()` to recycle placeholder bitmaps
  - Clear `mPendingUIUpdate` reference

---

## [1.1.4] - Aug 20, 2026

### Fixed
- **Android 14+ Foreground Service Permission Crash**: Resolved an issue where the app would crash immediately upon launch on Android 14 and 15 devices with the following error:
  ```
  SecurityException: startForeground requires android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK
  ```
  While the permission was declared in the manifest, it was never requested at runtime, causing MusicService to fail when attempting to start foreground playback on Android 14+ (API 34+).

### Enhanced Permission Management
- Added `FOREGROUND_SERVICE_MEDIA_PLAYBACK` to the runtime permission request flow on Android 14+ (API 34+)
- The app now properly requests all required permissions on first launch

### Technical Improvements
- **Permission Management**
  - Added `FOREGROUND_SERVICE_MEDIA_PLAYBACK` to runtime permission list
  - Proper permission handling for foreground media services

---

## [1.1.3] - Aug 19, 2026

### Fixed
- **Temporary Toast "Stuck" Issue in FolderManager**: Resolved an issue where the "Imported: [folder name]" toast would appear stuck instead of auto-dismissing after importing a folder via Storage Access Framework (SAF)
  - Temporary toasts now correctly auto-dismiss after their duration and are no longer reattached during activity lifecycle events

### Enhanced ToastManager Reliability
- Improved state machine with clear separation between temporary and persistent toast behavior
- Temporary toasts are now properly isolated from lifecycle events, preventing them from becoming stuck
- Persistent toasts continue to reattach correctly across configuration changes

### Technical Improvements
- **State Machine Enhancements**
  - `handleReattach()` now only processes persistent toasts, preventing temporary toasts from being incorrectly reattached
  - `onActivityResumed()` now distinguishes between temporary and persistent toasts, applying appropriate lifecycle handling for each
  - Added `mTemporaryToastDismissed` flag to prevent double-dismiss scenarios and ensure clean state transitions
  - Added `mTemporaryToastDuration` field for proper duration tracking and management

---

## [1.1.2] - Aug 10, 2026

### Fixed
- **Stuck Toasts**: Redesigned ToastManager now implements per-activity overlay management. This prevents toasts from getting orphaned during activity transitions
- **Overlay Leaks**: Activity lifecycle callbacks now properly clean up overlays when activities are destroyed
- **Multiple Overlay Stacking**: Strict single-toast policy enforced by state machine

### Technical Improvements
- **State Machine Architecture**
  - Enhanced 6-state machine with thread-safe atomic transitions
  - Event-driven state management with validation per state
  - Clear visibility into toast lifecycle for debugging
- **Heartbeat System Refinement**
  - Operation ID matching ensures only the correct operation can manage its own toast
  - Prevents unauthorized toast manipulation
- **Performance**
  - Optimized overlay lookups with activity-specific keys
- **Weak References** prevent memory leaks during configuration changes

---

## [1.1.1] - Aug 9, 2026

### What's New
- **Unified Playback Control**: The Play/Pause button now uses a single source of truth in MusicService, ensuring consistent behavior whether you tap the play button, use your headset controls, or interact with the notification. This eliminates the bug where playback would restart from the beginning instead of resuming from the paused position.
- **Enhanced Sort Button**: The SORT button in PlaylistActivity now features the same responsive press animations as the CANCEL and SEARCH buttons, creating a cohesive interaction experience when cycling through the 10 sort modes.
- **Consistent Equalizer Buttons**: The ENABLE/DISABLE, Preset and RESET buttons in the EqualizerActivity now match the press animation behavior found throughout the rest of the app, providing consistent tactile feedback across all buttons.
- **Faster Repeat/Shuffle Button Response**: Repeat and Shuffle buttons now respond faster with a reduced debounce delay of 250ms, providing more responsive feedback when cycling through modes.
- **Reliable Toast Lifecycle**: The ToastManager now properly transitions to the HIDDEN state after removing overlays, preventing toasts from getting stuck on screen. The state machine ensures toasts are always cleaned up correctly, even during activity transitions.

### Bug Fixes
- **Fixed Playback Resume Issue**: Playback now correctly resumes from the paused position instead of restarting the song from the beginning when tapping the play button
- **Fixed Stuck Toasts**: Toast messages in MusicPlayer no longer remain stuck on screen after their duration expires
- **Fixed Slow Repeat/Shuffle Response**: Reduced debounce delay from 500ms to 250ms for faster response and feedback

### Technical Improvements
- **Single Source of Truth**: `MusicService.togglePlayPause()` now handles all edge cases including empty playlist, invalid index, player not ready, and stuck transition recovery
- **Enhanced Player State Management**: Added explicit player readiness checks and saved position restoration for seamless resume
- **Improved Toast State Transitions**: `hideInternal()` now always transitions to HIDDEN state, eliminating stuck toast scenarios
- **Optimized Debouncing**: Introduced `REPEAT_SHUFFLE_DELAY_MS = 250` specifically for repeat and shuffle buttons

---

## [1.1.0] - Aug 2, 2026

### What's New
- **Genre Loading**: Songs now display their true metadata genres from ID3 tags. Genres load progressively in the background, updating the playlist in real-time as metadata is extracted from your music files.
- **Sort Modes**: With genres properly loaded, Genre A-Z and Genre Z-A are now included in the sort modes, allowing you to organize your playlist by music genre.
- **Playback Recovery System**: The app now automatically detects and recovers from playback stalls. If your music stops playing unexpectedly, Llama attempts recovery and resumes playback seamlessly.
- **Battery Optimization Request**: On Android Marshmallow and above, Llama requests exemption from battery optimizations, ensuring reliable background playback during extended screen-off periods.
- **Incoming File Improvements**: When opening audio files from other apps, the playback experience is now smoother and more reliable.

### Enhancements
- **Improved Scrolling Performance**: Playlist scrolling optimized with DiffUtil and payload-based updates for smoother navigation through large libraries.
- **Enhanced Notifications**: Better sync state management with a timeout watchdog to prevent stuck "Syncing" states.

### Technical Improvements
- **Database Version 2**: Added `file_size` and `uri` columns to `loose_songs` table for better consistency with the main `songs` table.
- **StorageObserver Auto-Sync**: Llama automatically detects new music files in your added folders and updates the playlist in the background without interrupting playback.
- **State Machine for Toasts**: ToastManager now uses a robust state machine to prevent toast duplication and ensure reliable overlay lifecycle management.
- **Enhanced Thread Safety**: MusicLibrary updated with comprehensive read-write locks for thread-safe song list access.
- **Batched Genre Updates**: Genre updates are batched to prevent scroll jitter during background loading.

### Removed
- **Removed SoundTouch library**: Llama now uses PlaybackParams for audio processing (Android 6.0+ required)

### System Requirements changed
- **Minimum SDK raised from Android 4.4 (KitKat) / API 19 to Android 6.0
  (Marshmallow) / API 23.** The SoundTouch native library was the reason
  for the API 19 floor. `PlaybackParams`, which replaces it, was
  introduced in API 23, so the floor moves with the replacement. The
  bottom of this file reflects the new minimum from this release onward.

### Bug Fixes
- Fixed issue where playlist sort mode changes in PlaylistActivity would not correctly sync to MusicService
- Fixed issue where metadata editor would lose current song's genre on app restart
- Fixed issue where notification channel name would not update properly when sync state changed
- Fixed issue where loose tracks would not appear in correct sort order
- Fixed issue where double-clicking Play button would cause UI glitches
- Fixed issue where FolderManager folder count would not update correctly after removing folders
- Fixed issue where "Loose Tracks" entry showed incorrect song counts after sync
- Fixed issue where album art would flash when returning to MusicPlayer from other activities
- Fixed issue where equalizer would not attach to audio session after app resume
- Fixed issue where MetadataEditor would crash when editing a file without write permission
- Fixed issue where sort button in PlaylistActivity would not update header display after theme changes

### Backward Compatibility
- **Android 14+ (API 34+)**: Added `FOREGROUND_SERVICE_MEDIA_PLAYBACK` permission for Google Play compliance
- **Android 13+ (API 33+)**: Added `POST_NOTIFICATIONS` and `READ_MEDIA_IMAGES` permissions
- **Android 12+ (API 31+)**: Added `BLUETOOTH_CONNECT` permission for headset controls

---

## [1.0.0] - May 27, 2026

### Initial Release

### Features
- **Pitch Control**: Adjust pitch from -6 to +6 semitones without affecting speed
- **Tempo Control**: Change playback speed from 0.5x to 1.5x without affecting pitch
- **10-Band Equalizer**: 60 preset configurations including custom preset (adjustable)
- **Bass Boost & Surround**: 0-100% adjustable effects
- **Metadata Editor**: Edit title, artist, album, genre, year, track, lyrics, and album art
- **Folder Management**: Import music folders via Storage Access Framework
- **Loose Tracks**: Play individual files from other apps
- **Search**: Live search across all metadata (128 character limit)
- **Sort Modes**: 10 sort options including title, artist, album, genre, and date
- **Theme Colors**: 30 color options
- **Bluetooth Support**: Headset connection detection and media buttons
- **Background Playback**: Notification controls and foreground service

### Known Issues
- SoundTouchPlayer implementation doesn't work although the native libraries are compiled successfully during build.

### System Requirements
- **Minimum SDK:** Android 4.4 (API 19)
- **Target SDK:** Android 14 (API 34)
- **Recommended RAM:** 512 MB or higher

---

## System Requirements

- **Minimum SDK:** Android 6.0 (API 23)
- **Target SDK:** Android 15 (API 35)
- **Recommended RAM:** 1 GB or higher

## Download

Download the APK from the Assets section of the latest release.

## Installation

1. Download the APK file
2. Enable "Unknown Sources" in your device settings
3. Open the APK file and tap "Install"

## First Time Setup

1. Tap the MANAGER button in the top right corner
2. Tap IMPORT FOLDER to select your music folder
3. Tap UPDATE to save your selection
4. The app will scan your folder and build the playlist

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

## Credits

- jaudiotagger by jthink Ltd
- FontAwesome by Fonticons, Inc.

## License

Licensed under the MIT License

Copyright (c) 2026 Richard Korbla Adzido

Made with love in Ghana