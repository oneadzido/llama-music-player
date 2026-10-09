# Changelog

All notable changes to Llama Music Player are documented in this file.

The sections at the bottom of this file — System Requirements, Download,
Installation, First Time Setup, How to Use, Credits, and License —
describe the current release and are not repeated per entry. A release
that changed one of those facts says so inside its own entry.

## [1.2.7] - Oct 9, 2026

### Overview

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

### Fixed

- **Removing a folder no longer wipes the playlist** — The songs
  table holds song paths: strings of the form
  `/storage/emulated/0/Music/song.mp3`. The selected-folder set
  holds folder URIs: strings of the form
  `content://com.android.externalstorage.documents/tree/primary%3AMusic`.
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
  table against the paths it found under the kept folders: it finds
  every song under every remaining folder and removes any row whose
  path it did not find. When the user has staged the Loose Tracks
  entry for removal, the loose-songs table is cleared before the
  scan runs, so the scan's reconciliation leaves the songs table
  holding exactly the songs under the kept folders. The scan
  operates on song paths, which are the values the songs table
  stores, so its reconciliation is the single correct write path for
  the songs table.

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
  on, the wait is retargeted to the newest path through the same
  gate. The cross-fade still begins only when the newest pending
  path's art resolves.

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
  between the bind and the callback — `loadFullPlaylistInBackground`
  would never run, `onLoadComplete` and `onLoadError` would never
  fire, and the heartbeat would re-post itself forever. The stuck
  detector watched the heartbeat and saw a live operation; it
  watched the overlay and saw a valid holder. Neither of its two
  stuck conditions could fire.

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

### Under the Hood

- `FolderManager.performPlaylistUpdate()`'s completion callback no
  longer clears the loose-songs table and no longer calls
  `SongDatabase.deleteSongsNotInSet` with the selected-folder set.
  The loose-songs table is cleared once, in `saveChanges`, before
  the update is dispatched; the songs table is reconciled once, by
  `PlaylistManager.performScan`, against the song paths the scan
  found under the kept folders. Each table has one write path for
  the update cycle.

- `AlbumArtCarousel.SwipeState` gained `REVEAL_PENDING`. The state
  is entered when a reveal is deferred because the incoming path's
  art has not yet resolved. While in this state the outgoing art
  stays on screen; no overlay is created and `render()` is not
  called for the incoming window, so the strip's middle slot cannot
  be written with the placeholder mid-transition.

- `AlbumArtCarousel` gained `mPendingRevealPath`, holding the path
  the pending reveal is waiting for, and
  `mPendingRevealTimeoutRunnable`, the safety-net timeout that fires
  when the art never resolves. Both are cleared by `cancelSwipe`,
  and `mPendingRevealPath` is retargeted by `setWindow`'s
  `REVEAL_PENDING` branch, which routes the retarget through
  `startReveal(String)` so the new path gets its own wait: the
  cross-fade starts immediately when the new path's art is already
  cached, and a fresh safety-net timeout is armed when it is not.

- `AlbumArtCarousel` gained `mDecodedPaths`, a set of paths whose
  decode has resolved — whether or not a bitmap was produced. A
  path in this set with no bitmap in the shared cache has been read
  and confirmed to have no embedded art.

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
  message, and the incoming-file hand-off is skipped because there
  is no service to hand it to.

- The three strings the abandon and refusal paths display —
  `MESSAGE_LIBRARY_LOAD_FAILED` and `MESSAGE_SERVICE_BIND_FAILED` —
  are private constants on `MusicPlayer`.

## [1.2.6] - Oct 8, 2026

### Overview

Three changes ship in this release — a unified animation source for
the transport buttons, a stable shuffle neighbour for the carousel,
and a reduced main-thread cost for the notification commit.

Every transition — a transport button, a headset key, an automatic
advance, a committed swipe, an incoming file — reaches the album-art
carousel through one signal: the service's `UPDATE_PLAYER` broadcast.
The broadcast carries the window the carousel renders, the direction
of the transition that produced it, and whether the transition
restarts the current song. The carousel renders exactly the animation
those three values describe. The strip's first frame and the audio's
transition are the same event, on the same main-thread turn, against
an idle main thread.

Under shuffle, the neighbour the carousel renders and the target the
next commit advances to are the same value by construction. The two
peek functions hold independent memos, so both directions' answers
survive a window read, and a subsequent commit that consumes either
direction returns the index the window advertised in that slot. The
service's window producer is also memoised for the current state, so
two consecutive reads with the same state return the same three
paths.

The notification commit's main-thread work is the builder
construction and the notification IPC. The controller caches its
themed action icons — three of the four use constant tints and are
identical across every commit, and the fourth depends only on the
theme color — and caches the content, close, and delete intents.

### Fixed

- **Transport buttons animate identically to automatic transitions**
  — A next or previous press is a plain request: it asks the service
  to advance, and the carousel's slide begins on the turn the
  service's broadcast reaches the activity. The service names the
  direction and the restart of every transition it begins, as
  properties of the generation that produces the window, and
  publishes both on the `UPDATE_PLAYER` broadcast that carries the
  window. The carousel reads them from the broadcast and renders the
  animation they describe. There is no prediction for the service to
  contradict and no second broadcast for the carousel to absorb. The
  reveal is reachable from a button press under shuffle.

- **The shuffle neighbour slots are stable** — Under shuffle, the
  neighbour the carousel renders and the target the next commit
  advances to are the same value. The two peek functions hold
  independent memos, so both directions' answers survive a window
  read; previously a single shared memo was overwritten by whichever
  peek ran last, and a subsequent commit that consumed the other
  direction's peek resolved to a fresh random index rather than the
  one the window showed. The problem was most visible when the
  shuffle history was exhausted, where the previous peek falls back
  to a fresh random pick: the window's previous slot and a Previous
  press would resolve to two different random indices, and the left
  slot would show one path's placeholder while the animation settled
  on another. The service's window producer is also memoised for the
  current state, so two consecutive reads with the same state return
  the same three paths.

- **The transport restart nudge is preserved** — Under repeat-one, on
  a single-song playlist, or on a shuffle re-pick that cannot avoid
  the current index, the service names the transition's direction and
  sets its restart flag. The carousel adopts the incoming window and
  runs the same below-threshold nudge the automatic restart runs. The
  two paths share the spring force, the outward distance, and the
  `RESTARTING` branch of `setWindow`, so a press and an involuntary
  restart look the same. Both legs of the nudge run on the same
  spring constants, so the out-move and the return share their motion
  model.

- **The notification commit is bounded by the notification IPC** —
  The four themed action icons and the three `PendingIntent` objects
  are served from caches. The main-thread work a commit performs is
  the notification builder construction and the notification IPC.

### Under the Hood

- `MusicService` names the direction and the restart of every
  transition as properties of the generation. Four fields carry them:
  `mGenerationDirection` and `mGenerationRestart` hold the values of
  the current generation, and `mNotedDirection` and `mNotedRestart`
  hold the values noted for the next transition to begin.
  `noteTransition(int, boolean)` writes the notes;
  `beginTransition(int, int)` consumes them into the generation it
  creates; `broadcastUpdate()` publishes them alongside the window;
  `clearNotedTransition()` clears them when `reconcile()` decides no
  transition is warranted.

- Every transition entry point notes the direction and restart at the
  moment it decides the target. `playNextSong()` and
  `playPreviousSong()` note `+1` and `-1` respectively, with the
  restart flag set when the request resolves to a restart.
  `onCompletion()` notes `+1`, with the flag set when the automatic
  advance restarts the same song under repeat-one or on a
  single-song playlist. `onStall()` notes `+1` for an advance and
  `0` for a reprepare of the same song. `onPlayerError()`,
  `onPrepareFailed()`, and the three failure paths inside
  `beginTransition` note `+1` and clear the restart flag. The
  explicit-selection entry points — `playSong()`,
  `playSongAtPosition()`, `playSongByPath()`,
  `playPendingSongIfExists()` — do not note; their transitions carry
  direction `0` and the carousel runs the reveal when the song
  changes.

- `MusicService.broadcastUpdate()` publishes the two values as
  `transition_direction` and `transition_restart` extras on the
  `UPDATE_PLAYER` broadcast. They travel with the window and the
  generation, so a receiver that has the window has the transition
  that produced it. `transitionToIdle()` and `stop()` clear both the
  current and the noted values, so an abandoned request does not
  leave a direction for a later, unrelated transition to pick up.

- `MusicService` holds two shuffle peek memos, one per direction.
  `mShuffleNextPeekValid`, `mShuffleNextPeekFromIndex`, and
  `mShuffleNextPeekResult` hold the next peek's answer;
  `mShufflePrevPeekValid`, `mShufflePrevPeekFromIndex`, and
  `mShufflePrevPeekResult` hold the previous peek's answer. The two
  memos have independent lifetimes: a window read that computes both
  peeks preserves both, so a commit that consumes either direction
  returns the index the window advertised in that slot. A single
  shared memo would be overwritten by whichever peek ran last, and a
  commit that consumed the other direction's peek would resolve to a
  fresh random index rather than the one the window showed.

- `MusicService.getWindowSnapshot()` memoises the window for the
  current `(generation, current path)` pair. The two peek functions
  are not idempotent on their own — each one recomputes the neighbour
  the corresponding memo returns — so two consecutive calls with the
  same state would produce two windows with different neighbour paths
  in the shuffle fallback case. The cache holds the window the last
  call produced, so every consumer in the same state sees the same
  three paths. This is what keeps the neighbour slot the carousel
  renders stable across the broadcast that produced it and any
  refresh that follows, and what keeps the window's neighbours in
  agreement with the shuffle picks the next commit will consume. The
  cache is cleared by `clearShufflePeek()`, which every operation
  that changes what an index means already calls, and by
  `transitionToIdle()` and `stop()`.

- `AlbumArtCarousel.setWindow(PlaybackWindow, int, boolean)` reads
  the two values alongside the window. Its `adoptInIdle` method uses
  the direction directly — `startCommit(int, boolean)` for a
  neighbour advance, `startBelowThreshold(int)` for a restart,
  `startFade()` when the direction is zero and the current song
  changed — and snaps to rest with no animation when the generation
  is unchanged, the current path is empty, or the direction is zero
  and the song is unchanged. The service's answer is the only source
  of the direction.

- Every motion the carousel runs toward a rest position runs on one
  spring: the restart nudge's outward leg, the restart nudge's return
  leg, and every return from a partial commit — the spring-back after
  a below-threshold swipe, the refusal path of a committed swipe, the
  mismatch path of an in-flight commit, the snapshot timeout. The
  damping ratio and stiffness are shared, so every settle has the
  same time to come to rest and the same small overshoot regardless
  of the distance travelled. The strip's motion spreads across the
  whole spring settle, so the arrival at rest is gentle.

- `MusicService.playNextSong()` and `MusicService.playPreviousSong()`
  are the transport entry points. They assign the target, note the
  direction and restart, and call `reconcile()`. The service
  publishes the transition alongside the window on the
  `UPDATE_PLAYER` broadcast, so a caller that requests a change does
  not need a second return channel.

- `MusicPlayer`'s `UPDATE_PLAYER` receiver reads
  `transition_direction` and `transition_restart` from the broadcast
  and forwards both to the carousel's `setWindow`. The
  `refreshCarouselFromService()` and `applyNoSongState()` callers
  pass `0` and `false`, because a re-publication is not a fresh
  transition.

- `MusicPlayer.playNextSong()` and `MusicPlayer.playPreviousSong()`
  are plain requests. A button press dispatches the service request
  on the tap and the strip starts moving on the turn the service's
  broadcast reaches the receiver — the same turn an automatic
  transition's slide starts on.

- The swipe commit listener installed in `MusicPlayer.initializeViews`
  routes a committed swipe through the same two methods. A swipe's
  commit slide is a continuation of the drag and begins at the
  release, because the strip is already displaced under the finger;
  the service's broadcast arrives during the slide and is absorbed
  by the commit state.

- `NotificationController` holds `mThemedIconCache`, a map keyed by
  the pair of the drawable resource and the tint, and three cached
  `PendingIntent` fields for the content, close, and delete intents.
  `getThemedIcon(int, int)` builds on a miss and serves from the
  cache on every subsequent call; `getContentIntent()`,
  `getCloseIntent()`, and `getDeleteIntent()` build on first use and
  return the cached instance thereafter. The theme-color dependency
  of the play/pause icon is expressed through the key: a theme change
  alters the tint, which alters the key, which produces a fresh entry
  on the next commit alongside the old. The cache is cleared in
  `shutdown()`.

- `NotificationController.buildPlaybackNotification` reads its four
  action icons and its three intents through the getters. The
  `buildContentIntent(int flags)` helper is folded into
  `getContentIntent()`, since every caller reads the cached instance.
  `buildLibraryNotification` reads the content intent through the
  same getter.

---

## [1.2.5] - Oct 7, 2026

### Overview

Four changes ship in this release — a Material Design 3 motion system
for the album-art carousel, a unified commit turn that lands every
surface of the app from one frame, a single-song playlist resolution
for the transport buttons, and a preservation pass over the
incoming-file pathway.

The album-art carousel now uses the platform's motion tokens. The
commit slide runs on the emphasized decelerate curve over 500 ms. The
non-neighbour reveal — which is what an incoming file opened from
another app produces — is a staggered cross-fade: the outgoing art
fades out on the emphasized accelerate curve over the first 45 % of a
600 ms window, and the incoming art fades in on the emphasized
decelerate curve with a 0.96→1.0 scale over the remaining 55 %,
overlapping the outgoing leg by 15 % of the total duration. The
repeat-one nudge is driven by a `SpringAnimation` with a low-bouncy
damping ratio, so the strip overshoots its target slightly and
settles back with a short oscillation — a physical acknowledgement
of the restart rather than a linear bounce.

Every transition the user can cause now commits every surface of the
app from one main-thread frame. The carousel's `setWindow` method
reports whether the window landed synchronously, and the activity's
`UPDATE_PLAYER` receiver builds one commit runnable that writes the
notification, the media session, the transport glyph, the seek bar's
range, the title, and the artist. When the carousel is animating, the
runnable runs on the settle frame; when it is not, the runnable runs
in the receiver's turn. Either way, the album art and the text land
on the same frame.

On a single-song playlist, the next and previous buttons now restart
the song on every repeat mode. Previously they did nothing under
repeat-off and repeat-all: the wrap landed on the current index, the
target did not change, and the state machine did not transition. The
service exposes two pure methods, `willNextRestartCurrent()` and
`willPreviousRestartCurrent()`, and the activity uses the answer to
choose between the carousel's below-threshold nudge and the full
commit slide.

Two rapid incoming files now resolve in order even when their inserts
overlap, and a request that arrives during an in-flight transition is
no longer lost to a failure-recovery heuristic. Three related gaps
are closed: the insert's completion no longer re-fires the hand-off
that parks the pending path, the service's `setPlaylist` no longer
falls back to index 0 when the pending path is not yet findable, and
the service gained a `mDeferredReconcile` flag that preserves the
newest target through a prepare failure.

### Fixed

- **The album art and the text land on the same frame** — Previously
  the `UPDATE_PLAYER` receiver wrote the seek bar, the title, the
  artist, and the transport glyph immediately while the carousel was
  still animating. Those writes triggered a measure-and-layout pass
  that landed on a frame between the animation's own frames, which
  the eye read as a stutter in the slide and as the art landing
  before the text. The receiver now builds one commit runnable and
  dispatches it through the carousel's commit contract, so the work
  and the carousel's own re-render land in the same layout pass.

- **The next and previous buttons no longer schedule a delayed glyph
  update** — Both methods scheduled
  `postDelayed(updatePlayPauseButton, 100)`, which landed a layout
  pass on a mid-slide frame. The glyph is delivered by the
  `UPDATE_PLAYER` receiver through the carousel's settle hook, on
  the same frame the strip's new bitmaps are installed. The delayed
  call and the `NEXT_PREV_UPDATE_DELAY_MS` constant are gone.

- **The album-art carousel uses Material Design 3 motion** — The
  commit slide, the reveal, and the nudge now use the platform's
  motion tokens. The commit slide runs on the emphasized decelerate
  curve (`PathInterpolator(0.05, 0.7, 0.1, 1.0)`). The reveal is a
  staggered cross-fade: the outgoing art fades out on the emphasized
  accelerate curve (`PathInterpolator(0.3, 0.0, 0.8, 0.15)`) over
  the first 45 % of a 600 ms window, and the incoming art fades in
  on the emphasized decelerate curve with a 0.96→1.0 scale over the
  remaining 55 %, overlapping the outgoing leg by 15 % of the total
  duration. The nudge is driven by a `SpringAnimation` with a
  low-bouncy damping ratio (0.75) and low stiffness (200.0), so the
  strip overshoots its target slightly and settles back with a
  short oscillation. The reveal's overlay is composited on a
  hardware layer for the duration of the fade, cleared on the end
  frame.

- **Single-song playlist transport buttons restart the song** — On
  a playlist with one song, the next and previous buttons did
  nothing under repeat-off and repeat-all: the wrap landed on the
  current index, the target did not change, and the state machine
  did not transition. The buttons now restart the song on every
  repeat mode. `willNextRestartCurrent()` and
  `willPreviousRestartCurrent()` answer the activity's question
  about the intent, and the carousel's nudge now accepts a
  neighbour whose path equals the current path, so the restart has
  somewhere to slide.

- **A request that arrives during an in-flight transition is not
  lost** — The service's `reconcile()` bails when the state is
  PREPARING, and the only re-entry point was a comparison of the
  target index against the current index in the prepare callbacks.
  That comparison does not fire when a failure-recovery heuristic
  has already reset the target to a positional successor. A new
  `mDeferredReconcile` flag is set on every PREPARING bail, cleared
  before a new transition starts, and checked by both prepare
  callbacks. A request that arrived while a transition was in
  flight re-enters reconciliation with the newest target even
  after a prepare failure.

- **A rebuild during an in-flight transition no longer redirects
  the target** — When the state is PREPARING and a playlist
  rebuild runs, the rebuild used to reset `mTargetIndex` from the
  anchor. By that point the pending path had already been consumed
  by the successful resolution of the first request, so the anchor
  was the currently bound song, and the rebuild silently
  redirected the transition to it. The rebuild now re-resolves the
  target's path against the new playlist, so the same song is
  named and the transition in flight is preserved.

- **Two rapid loose-track opens resolve in order** — When a user
  opens two audio files in quick succession, each file is inserted
  as a loose track and its own playlist rebuild follows.
  Previously `MusicService.setPlaylist` cleared `mPendingSongPath`
  unconditionally, so the first rebuild — running before the
  second file's insert had committed — cleared the slot even
  though the path it named was not yet in the playlist. The second
  rebuild fell through to the persisted last-played path, and the
  second file never played. The slot is now consumed only when it
  was resolved by its own path in the rebuilt playlist, and the
  insert's completion no longer re-fires the hand-off, so a newer
  request is not overwritten by an older one.

- **The notification surface no longer drifts from the media
  session** — Three fixes. The rendered-state advance is now gated
  on the notification post actually landing, so a revoked
  permission or a failed IPC rolls back the rendered state and
  retries on the next sync. Art retention across a re-render of
  the same song is gated on the decode still being in flight, so
  a resolved "no art" clears both surfaces together. And a theme
  change re-applies the session metadata from the last committed
  snapshot, so the session's placeholder picks up the new tint on
  the same turn as the notification's.

### Under the Hood

- `AlbumArtCarousel` gained the Material Design 3 motion constants
  (`EMPHASIZED_DECELERATE`, `EMPHASIZED_ACCELERATE`,
  `NUDGE_SPRING_DAMPING_RATIO`, `NUDGE_SPRING_STIFFNESS`,
  `COMMIT_DURATION_MS = 500`, `RETURN_DURATION_MS = 350`,
  `REVEAL_DURATION_MS = 600`,
  `REVEAL_OUTGOING_FRACTION = 0.45`,
  `REVEAL_INCOMING_START_SCALE = 0.96`,
  `REVEAL_OVERLAP_FRACTION = 0.15`). The commit slide, the
  return-to-rest glide, and the reveal's incoming leg use the
  emphasized decelerate curve; the reveal's outgoing leg uses the
  emphasized accelerate curve. The below-threshold nudge is driven
  by a `SpringAnimation` with a low-bouncy `SpringForce`. The
  reveal's overlay is composited on a hardware layer for the
  duration of the fade. `setWindow(PlaybackWindow)` and
  `adoptInIdle(PlaybackWindow, long)` now return a boolean — the
  commit contract — and `isAnimating()` reports true while
  `mReturningToRest` is set, so a return-to-rest glide is honestly
  reported to callers that gate work on the strip's motion.
  `nudgeToNeighbour(int)` accepts a neighbour whose path equals the
  current path, so a single-song playlist's restart has somewhere
  to slide. The `SWIPE_ANIMATION_MS` constant and the two
  interpolator imports were removed.

- `MusicPlayer`. The `UPDATE_PLAYER` receiver adopts the window,
  then dispatches one commit runnable through the carousel's commit
  contract. `playNextSong()` and `playPreviousSong()` ask the
  service through `willNextRestartCurrent()` and
  `willPreviousRestartCurrent()` and choose the carousel's nudge or
  commit accordingly. `insertLooseTrackAndHandOff`'s completion
  callback no longer re-fires `handOffIncomingPath` — the initial
  hand-off in `handleIncomingFile` is the only writer of the
  pending slot. The `mUpdatePlaylistReceiver` and
  `mUpdatePlaylistFromFoldersReceiver` no longer call
  `refreshCarouselFromService`; the carousel is updated by the
  service's own `UPDATE_PLAYER` broadcast, so a second path cannot
  show the target song's art while the previous song is still
  playing. The `postDelayed(updatePlayPauseButton, 100)` calls and
  the `NEXT_PREV_UPDATE_DELAY_MS` constant were removed.

- `MusicService` gained `mDeferredReconcile` and the two pure
  methods `willNextRestartCurrent()` and
  `willPreviousRestartCurrent()`. `reconcile()` sets
  `mDeferredReconcile` when it bails on PREPARING and clears it
  before `beginTransition`. `onPrepared` and `onPlaybackStarted`
  re-check on `mDeferredReconcile || mTargetIndex != mCurrentIndex
  || mForceReprepare`. `onPrepareFailed` respects the deferred flag
  and re-enters reconciliation with the newest target instead of
  overwriting it with a positional successor. `playNextSong` and
  `playPreviousSong` force a reprepare when the wrap lands on the
  current index, matching what the automatic advance already does
  at the end of a track. `setPlaylist` captures the target path
  before the playlist swap, preserves the target through a rebuild
  that runs while the state is PREPARING, and returns without
  falling back to index 0 when the anchor is the pending path and
  the path was not found. `broadcastUpdate` pokes the notification
  controller after dispatch. `transitionToIdle` and `stop` clear
  the deferred flag.

- `PlaybackWindow`. The Null Semantics paragraph was corrected to
  describe what the service actually produces: null neighbours only
  when the playlist is empty; the current path as its own neighbour
  on a single-song playlist or an unavoidable shuffle re-pick.

- `NotificationController` gained `commitNow()`, which runs one
  sync synchronously on the calling main-thread turn; the activity
  calls it from the carousel's settle frame. `syncOnce` advances
  `mLastRenderedSnapshot` only when the post landed. `applySnapshot`
  returns a boolean and rolls back `mPlaybackState` on a failed
  post. `performPlaybackUpdate` returns a boolean. Art retention
  across a re-render of the same song is gated on
  `!snapshot.artResolved`. `onThemeChanged` re-applies the media
  session metadata from the last committed snapshot before
  rebuilding the notification.

- The commit slide and the return-to-rest glide use
  `COMMIT_DURATION_MS = 500` and `RETURN_DURATION_MS = 350`
  respectively, both on the emphasized decelerate curve. The
  reveal uses `REVEAL_DURATION_MS = 600` split into an outgoing
  leg of 45 % and an incoming leg of the remaining 55 % with a 15 %
  overlap. The nudge uses the platform's
  `SpringForce.DAMPING_RATIO_LOW_BOUNCY` and
  `SpringForce.STIFFNESS_LOW` constants.

---

## [1.2.4] - Oct 6, 2026

### Overview

Two fixes ship in this release: one in the player's split-screen
layout, one in the incoming-file pathway. A supporting change to the
toast position follows the split-screen fix because the two are
inseparable: the same padding reduction that makes room for the
artist line moves the gap the toast is centered in.

In split-screen mode, when the album art hides to make room for the
title, artist, seek bar, and controls, the title and artist group now
moves up to occupy the space the album art vacated. Previously a
padding band survived above the group after the album art was hidden,
and the artist line was pushed past the visible portion of the window.
The wrapper is now hidden together with the album-art viewport, the
group is pinned to the top of its slot at the album-art top's Y
offset, and the transport row's bottom padding is reduced so the
group's weight slot gains the height the artist line needs. The toast
position follows the same reduction so the toast stays centered in
the visible gap between the transport row and the metadata row.

When a user opens an audio file from another app, the file is added
as a loose track, the playlist rebuilds from the database, and the
song plays through the same transition path a transport request
uses. Previously the loose-track insertion announced the change with
a broadcast no component listened for, so the service's playlist was
never rebuilt, the pending song path was never resolved, and the
song appeared in the folder list but not in the playlist. The
insertion now announces the change with `UPDATE_PLAYLIST`, the
vocabulary every actor in the process already listens for.

### Fixed

- **Title and artist fill the space the album art vacates in
  split-screen** — When the window dropped below the minimum height
  that supports the album art, the album-art viewport was set to
  `GONE`, but the wrapper around it kept contributing its top and
  bottom padding. That padding survived as a visible band above the
  song-info group, and the group's centre-gravity layout pushed the
  artist line past the bottom of the visible window. Two changes
  fix both halves of the problem.

  The wrapper is now hidden together with the viewport, so its
  padding is not laid out. The title-and-artist group is pinned to
  the top of its weight slot and given a top offset equal to the
  wrapper's declared top padding — the same offset the album-art
  top carries in full-screen mode — so the title lands where the
  album art used to begin.

  The transport row's bottom padding is reduced from 64 dp to 48 dp
  at the same time. The reduction passes 16 dp of slot height to
  the title-and-artist group, which is what the artist line needs
  in a short split window; without it the artist would extend past
  the group's bottom and be clipped. The reduced value keeps the
  transport buttons clear of the temporary volume toast, which
  sits at a fixed offset from the window bottom.

  Every view below the group — the seek bar, the transport controls,
  and the metadata row — keeps its position relative to the bottom
  of the window, because the group's weight slot absorbs the
  difference. Full-screen and shrunk-album-art layouts are
  unchanged: the wrapper, the group, and the transport row all
  return to their layout-declared gravity, padding, and margins.

- **Toasts stay centred in the visible gap in both window modes** —
  The toast is anchored to the window bottom by a fixed margin. That
  margin was tuned against the full-screen layout's gap between the
  transport row and the metadata row: 80 dp of visible space, with
  the toast centred at 76 dp. When the split-screen fix reduced the
  transport row's bottom padding, the gap shrank to 64 dp and the
  toast's centre moved eight dp off from where the smaller gap
  expects it. The mismatch read as a small vertical asymmetry.

  `ToastManager` now chooses its bottom margin from the window's
  current height. In a full window it uses the original 76 dp; at or
  below the threshold at which the player hides its album art, it
  uses 68 dp — the same value shifted by exactly half the padding
  reduction, so the toast follows the centre of the smaller gap. The
  choice is made from `Configuration.screenHeightDp`, which every
  activity sees for the same window, so the correct margin is used
  in every activity without any activity having to announce its own
  layout state. Only one number changed in `createOverlayInView`;
  the rest of the toast's presentation — padding, corner radius,
  elevation, fade, auto-dismiss, operation ownership — is untouched.

- **Incoming files play through the state machine and appear in the
  playlist** — Opening an audio file from another app added the file
  to the loose-tracks table and broadcast `SOFT_UPDATE_PLAYLIST`,
  which no receiver in the codebase handles. The service's playlist
  was therefore never rebuilt, the pending song path was never
  resolved, and the state machine was never asked to transition.
  The song appeared in the folder list — which reads the
  loose-tracks table — but not in the playlist — which the service
  owns — and it never played. The loose-track insertion now
  announces the change with `UPDATE_PLAYLIST`, the same broadcast a
  folder scan and every other playlist-changing operation already
  uses. The service's receiver refreshes its library, rebuilds its
  playlist, resolves the pending song path as the anchor, and drives
  the state machine to the new song. The incoming file is played
  through the same transition path a next or previous request uses.

  Every state the state machine can be in when the incoming file
  arrives is handled by the code the app already runs for every
  other playlist rebuild: playing, paused, `PREPARING`, `IDLE` with
  no current song, unbound service, file already in the library,
  file already a loose track, insert failure, and the case where the
  resulting playlist is empty.

  A related change accompanies the fix: a pending song path is now
  treated as an explicit play request. On a fresh session — no
  current song, the state machine in `IDLE` — the anchor song
  normally sits in `READY` waiting for the play button, because
  there is no prior playback signal to honour. An incoming file is
  different: the user asked for it to play. The pause-after-prepare
  is therefore not applied when the anchor came from the pending
  path, and the incoming file auto-plays rather than loading paused.

### Under the Hood

- `res/layout/activity_music_player.xml` gained three identifiers:
  `albumArtWrapper` on the album-art padding container,
  `songInfoGroup` on the title-and-artist group, and `transportRow`
  on the row that holds the shuffle, previous, play, next, and
  repeat buttons. Nothing else in the layout moved.

- `MusicPlayer.initializeViews()` resolves the three new views and
  captures the group's layout-declared gravity and top padding, and
  the transport row's layout-declared bottom padding.
  `MusicPlayer.applyAlbumArtSize()` calls a new
  `applySongInfoLayout(boolean)` after the album-art viewport's size
  and visibility have been applied. The new method hides or restores
  the wrapper, pins or restores the group's gravity and top padding,
  and reduces or restores the transport row's bottom padding.

- `TRANSPORT_ROW_SPLIT_BOTTOM_DP = 48` is the split-screen bottom
  padding of the transport row, sixteen dp less than the layout's
  declared value. The constant is separate from the layout's own
  value so the split-screen reduction is expressed in the activity,
  not in the layout, and the full-screen padding is never touched by
  the split-screen code path.

- `ToastManager` gained a `TOAST_BOTTOM_MARGIN_COMPACT_DP = 68`
  constant and a `COMPACT_LAYOUT_HEIGHT_DP = 400` threshold. The
  original `TOAST_BOTTOM_MARGIN_DP = 76` is kept unchanged.
  `ToastManager.resolveBottomMarginDp()` reads the window height from
  the application resources at toast-creation time and returns one of
  the two constants. `createOverlayInView` uses that value for the
  container's `bottomMargin` instead of the fixed constant. Every
  caller keeps its current signature; the choice is made in the
  manager, from a value every activity shares for the same window.

- `PlaylistManager.addLooseTrackAndUpdate` broadcasts
  `UPDATE_PLAYLIST` in place of the unhandled
  `SOFT_UPDATE_PLAYLIST`, ordered before `DISPLAY_PLAYLIST` so the
  playlist activity's refresh reads the rebuilt playlist rather than
  the pre-rebuild one. The `SET_PENDING_SONG` broadcast and the
  `MusicService.setPendingSongPath` call are unchanged; the service
  already resolves a pending song path as the preferred anchor during
  a playlist rebuild.

- `MusicService.setPlaylist(ArrayList<Song>, boolean)` gained one
  condition in its cold-start branch: the pause-after-prepare is
  applied only when the anchor came from persisted state, not when it
  came from a pending song path.

---

## [1.2.3] - Oct 4, 2026

### Overview

Six fixes ship in this release, all in the playback layer.

The music-stream volume receiver is now registered only after the user
has opened the player. Previously the receiver was armed as soon as
the service was created, including on paths where the service starts
without the user opening the app — a Bluetooth connect, a sticky
restart, a media button. On one of those paths the service could be
left holding a prepared song in the `READY` state, and the next
hardware volume press started playback. The receiver is now registered
on bind and unregistered on destroy, so it exists only after the user
has opened the player.

A transient audio-focus loss is now remembered. When another app takes
focus for a bounded interval — a phone call, a navigation prompt, a
WhatsApp video status — the service pauses and remembers that it did
so. When focus is returned, playback resumes from where it paused. A
permanent loss is still respected: the service pauses and does not
resume on its own.

An interruption that offers to share the output — a navigation
prompt, a notification tone, a voice assistant reply — now ducks the
music instead of pausing it. The music keeps playing underneath at a
lower output level and returns to full volume when the interruption
ends. The system volume slider does not move.

The volume-up key now resumes playback from the foreground even when
the volume is already at the device maximum. At the maximum, the
framework emits no `VOLUME_CHANGED_ACTION`, so the service receiver
never fires; the volume key press is intercepted at the activity
level instead.

Two supporting changes ride alongside. The volume handler now reads
the current music-stream volume at the handling moment rather than
trusting the value carried by the broadcast. The storage observer no
longer starts a playlist scan when a non-audio file is written.

### Fixed

- **The volume key no longer affects playback before the app has been
  opened** — The receiver was registered in `onCreate()`, so every
  path that created the service armed it. One of those paths —
  `handleBluetoothResume` — can leave the service holding a prepared
  song in `READY` without the user ever opening the player: it calls
  `setPlaylist(...)` and then `resume()` before the async prepare has
  completed, so `resume()` sees `PREPARING` and does nothing, and the
  state settles in `READY` when the prepare finishes. With the
  receiver armed, a later hardware volume press ran the `READY` →
  `PLAYING` transition and started music. The receiver is now
  registered in `onBind(Intent)` and unregistered in `onDestroy()`.
  Its presence is the session state. A service created without a
  bound client never registers it, so a hardware volume press on
  those paths does nothing.

  Once the user opens the player the receiver is armed, and the
  behaviour is preserved in every respect. On a cold start the
  anchor song sits in `READY` waiting for the play button, and a
  hardware volume press — or the album-art volume gesture — starts
  playback through the same transition the play button uses. The
  receiver stays registered for the life of the service, so
  background playback, headset keys, and the settings slider all
  continue to work exactly as before. The fix narrows *when the
  receiver exists*; it does not change *what the receiver does once
  it exists*.

- **Playback resumes when a transient audio-focus loss ends** — When
  another app requested focus with `AUDIOFOCUS_GAIN_TRANSIENT` — a
  phone call, a navigation prompt, a WhatsApp video status — the
  service paused but did not remember why, so when focus returned it
  stayed paused. The interrupting app's end was a silent event that
  left the song sitting where it had stopped. The service now sets
  `mPausedByTransientLoss` on a transient loss and, when
  `AUDIOFOCUS_GAIN` arrives while that flag is still set, resumes
  from the paused position. A permanent loss (`AUDIOFOCUS_LOSS`)
  does not set the flag: the user has moved to another app or
  another permanent audio source, and playback does not restart on
  its own. The flag is cleared by any user action — resume, pause,
  seek, stop, or a volume drop to zero — so a stale flag cannot
  resume playback after the user has moved on.

- **Music ducks instead of pausing when an interruption offers to
  share the output** — Some interrupting apps request focus with
  `AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK`. The framework delivers this
  to the losing app as
  `AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK`. A navigation prompt, a
  notification tone, or a voice assistant reply is what sends this:
  the interrupting app is willing to share the output at a lower
  level, and would rather the music keep playing underneath than
  pause and resume around the interruption. The service previously
  ignored this code, so the music kept playing at full volume while
  the interruption played over it. The service now lowers the
  MediaPlayer's own output volume to `DUCK_VOLUME_FRACTION = 0.2f`
  for the duration of the interruption, and restores it to full when
  `AUDIOFOCUS_GAIN` arrives. Playback is not paused and the position
  is not lost. The system stream volume is not touched, so the
  volume slider does not move and the music-stream receiver does not
  fire.

- **Volume up at the device maximum resumes from the foreground** —
  When the music volume is already at the device maximum and the
  user presses volume up, the framework emits no
  `VOLUME_CHANGED_ACTION`: no value changed, so no broadcast fires.
  The service's receiver therefore never runs, and the resume path
  was unreachable in that case. `MusicPlayer` now overrides
  `dispatchKeyEvent` to observe `KEYCODE_VOLUME_UP` before the
  window system consumes it. When the volume is at the device
  maximum and the song is bound but paused, the activity resumes
  the song directly. The event is not consumed, so the system still
  processes the key as it normally does.

  The activity path covers the foreground case only. The framework
  does not deliver volume keys to a backgrounded activity, and no
  broadcast fires at the device maximum because no value changed.
  Every non-intrusive background channel has the same limitation:
  `VOLUME_CHANGED_ACTION` fires on a change that does not occur,
  `ContentObserver` on the volume setting has the same problem, and
  the media button action carries `KEYCODE_MEDIA_*`, not
  `KEYCODE_VOLUME_*`. The framework-supported paths that would reach
  a backgrounded app are all heavier than the case warrants — an
  accessibility service, a system overlay window, or a `MediaSession`
  remote volume provider that would move volume control away from the
  system stream. When the player is backgrounded and the song is
  deliberately paused, resuming goes through the notification's play
  action, which is delivered to the service through the media session
  in every screen state. That is the platform's intended affordance
  for the case, and it works from the lock screen and from the shade.

- **Volume decisions read the current system volume** — Some devices
  deliver `VOLUME_CHANGED_ACTION` after a follow-up change has
  already landed. The broadcast's `EXTRA_VOLUME_STREAM_VALUE` extra
  then describes a value the user has already moved past, and the
  pause-or-resume decision was made from that stale value. The
  handler now reads `getStreamVolume(STREAM_MUSIC)` at the handling
  moment, clamps it to the range the platform reports, and falls back
  to the broadcast extra only when the read fails. The gate is
  unchanged: zero pauses, above zero resumes or starts.

- **Storage observer no longer fires on non-audio writes** —
  `StorageObserver` registered two content observers, one for
  `MediaStore.Audio.Media.EXTERNAL_CONTENT_URI` and one for
  `MediaStore.Files.getContentUri("external")`. The second fires for
  every non-audio file write on external storage, and its URI
  contains `external`, so the filter — `contains("audio") &&
  contains("external")` — could never drop it. Every image save,
  video write, or download triggered a quiet period and a playlist
  scan. The `Files` observer is removed, and the filter is now
  `contains("/audio/")`. Audio additions, removals, and metadata
  edits — including the app's own edits, which are flagged as self
  changes — still trigger the scan. Image, video, and download
  writes do not.

### Under the Hood

- `MusicService.registerMusicVolumeReceiver()` is called from
  `onBind(Intent)` and nowhere else. It is idempotent, so a repeat
  bind is a no-op, and a registration failure leaves
  `mMusicVolumeReceiver` null so the next bind retries.
  `MusicService.unregisterMusicVolumeReceiver()` in `onDestroy()`
  remains the single teardown path. There is no separate session
  flag: the receiver's own field is the session state, so there is
  nothing for a second representation to drift against.

- `MusicService.handleAudioFocusChange(int)` distinguishes four
  cases. A permanent loss pauses and does not set any resume
  intent. A transient loss pauses and sets
  `mPausedByTransientLoss`. A duck request sets `mIsDucked` and
  applies the duck volume to the current player.
  `AUDIOFOCUS_GAIN` clears both flags: it restores the player's
  output volume if a duck was active, and resumes from the paused
  position if a transient pause was remembered and the state is
  still `PAUSED`. The resume flag is cleared at the top of
  `doResume()`, at the top of `pause()`, in the `PLAYING` arm of
  `togglePlayPause()`, in the zero-target arm of
  `onMusicVolumeChanged(int)`, in the `KEYCODE_MEDIA_PAUSE` arm of
  `handleMediaButton(int)`, and in the `onPause` and `onStop`
  callbacks of the `MediaSessionCompat`. Every user-initiated
  transition through any of those paths supersedes a pending
  auto-resume.

- `MusicService.applyDuckVolume(MediaPlayer)` is the single point
  that reads `mIsDucked` and writes the resulting volume to a
  player. It runs on the player thread. It is called from
  `handleAudioFocusChange(int)` when a duck begins or ends, and from
  `beginTransition(int, int)` after each new MediaPlayer is created,
  so a track change that happens mid-duck does not momentarily
  restore the volume to full. The duck is a scale applied to the
  MediaPlayer's own output through `setVolume(float, float)`, not a
  change to the system stream volume, so the framework's
  `VOLUME_CHANGED_ACTION` does not fire and the music-stream
  receiver is not involved.

- `MusicService.isPausedOrReady()` returns true when the state
  machine holds a bound song that is not currently playing. It is
  the case a volume-key press starts playback from: the song is
  paused at a position, or prepared at its start and waiting for the
  play button. `MusicPlayer` reads it from the at-maximum volume-up
  path.

- `MusicPlayer.dispatchKeyEvent(KeyEvent)` observes
  `KEYCODE_VOLUME_UP` before the window system consumes it, and
  performs one side effect: when the volume is at the device maximum
  and the service reports `isPausedOrReady()`, the activity calls
  `MusicService.resume()`. The event is not consumed. Volume changes
  below the maximum are handled by the receiver in `MusicService`,
  which fires on the broadcast the system emits when the value
  changes. The activity path exists solely for the case where no
  value changes, and therefore no broadcast fires. It runs only
  while the activity is visible: the framework does not deliver key
  events to a backgrounded activity. The background resume path for
  a deliberately paused song is the notification's play action,
  which reaches `MediaSessionCompat.Callback.onPlay` and runs the
  same `doResume()` the activity path runs.

- `MusicService.onMusicVolumeChanged(int)` reads
  `getStreamVolume(STREAM_MUSIC)` at the handling moment and clamps
  it to `[0, getStreamMaxVolume(STREAM_MUSIC)]`. The broadcast extra
  is used only as a fallback when the read fails. The decision is a
  single test on the clamped target: zero pauses, above zero
  resumes or starts. It is not gated on a delta, on the value having
  changed, or on the new value being greater than the old one.

- `MusicService.onMusicVolumeChanged(int)` retains the `READY` branch
  introduced in 1.2.2. Raising the volume while the song is loaded
  and waiting in `READY` calls `togglePlayPause()`, which runs the
  `READY` → `PLAYING` transition on the player thread through the
  start-token guard and the `onPlaybackStarted` commit. Once the
  user has opened the player this case is reached exactly as it was
  in the previous release. The session gate introduced in this
  release does not affect it, because the receiver is registered as
  soon as the player binds.

- `StorageObserver.startObserving()` registers a single content
  observer, for `MediaStore.Audio.Media.EXTERNAL_CONTENT_URI`.

- `StorageObserver.handleChange(Uri)` filters on the `/audio/` path
  segment. Every timeout — `SYNC_COOLDOWN_MS`, `QUIET_PERIOD_MS`,
  `GENRE_LOAD_DELAY_MS`, `OPERATION_END_DELAY_MS` — is unchanged.
  The cooldown check, the update-in-progress check, the
  `selfChange` handling, and the quiet-period scheduling are
  unchanged.

---

## [1.2.2] - Oct 2, 2026

### Overview

The album-art volume gesture now starts playback from the moment
a song loads, not only after the first tap on the play button. The folder
list scrolls naturally in a short split-screen window instead of being
hidden. Two click-debounce timers that were still reading wall-clock
time have been moved to the monotonic clock the rest of the app uses.

### Fixed

- **Volume gesture works from the moment a song loads** — On a cold
  start, the service prepares the anchor song and waits for the play
  button before starting playback. The album-art volume gesture did
  not start playback in that window: raising the volume changed the
  stream volume but left the song sitting in `READY`. The gesture now
  starts playback through the same transition the play button uses, so
  raising the volume after a cold start is enough to begin listening.
  The rule stays symmetric with the paused case: a drop to zero
  pauses, a raise from zero resumes or starts.

- **Folder list stays visible in split-screen** — The folder list used
  to be hidden whenever the window dropped below a fixed height, on
  the theory that its rows could not be usefully compressed. The
  list's layout already handles a shorter window: its RecyclerView
  carries weight and scrolls, so hiding it solved a problem the layout
  had already solved. The list now stays visible and scrolls in every
  window size. The empty-state placeholder appears only when the list
  is actually empty, not as a response to the window shrinking.

### Under the Hood

- `MusicService.onMusicVolumeChanged` gained a `READY` branch in the
  non-zero arm. `READY` is the cold-start state: a song has been
  prepared and is waiting for the play button. Raising the volume
  calls `togglePlayPause()`, which already contains the correct
  `READY` → `PLAYING` transition on the player thread, the
  start-token guard, and the `onPlaybackStarted` commit. No new state
  and no new code path is introduced. `IDLE` and `STOPPING` remain
  no-ops because neither has a song bound.

- `FolderManager` no longer carries a split-screen branch. The
  `SPLIT_SCREEN_MIN_HEIGHT_DP` constant, the
  `applyFolderManagerSplitLayout(int)` method, and the call sites that
  invoked it from `onCreate` and `onConfigurationChanged` are removed.
  `onConfigurationChanged` now calls a new
  `updateLayoutForConfiguration()` method whose only decision is
  whether the adapter is empty: the empty-state placeholder is shown
  when it is, and hidden when it is not. Empty-state visibility is
  therefore determined solely by adapter item count, matching the
  design rule that XML owns layout and Java owns data.

- Two click-debounce timestamps that still used
  `System.currentTimeMillis()` now use
  `SystemClock.elapsedRealtime()`. The folder manager's cancel button
  and the equalizer's preset button were the last two call sites on
  the wall clock; every other debounce site in the app was already
  monotonic. A wall-clock timestamp can move backwards — on a
  daylight-saving transition, a manual clock change, or an NTP
  correction — and a backwards move can either suppress a legitimate
  click or admit a duplicate one. The monotonic clock is immune to
  both.

---

## [1.2.1] - Oct 1, 2026

### Overview

The album-art carousel now animates on every kind of song transition —
a track ending on its own, a headset media button, a shuffle pick, a
repeat-one restart — with the same visuals a swipe or a button press
produces. Repeat-one has a dedicated animation that matches its
behaviour: a small glide toward the neighbour and back, whether the
restart came from the user's press or the track looping on its own. The
playlist sort has been moved off the main thread in both the service
and the adapter, so the sort overlay's spinner rotates freely
throughout the operation, and the overlay now waits for the
RecyclerView to settle with the new ordering before it dismisses.

### What's New

- **Carousel animates on automatic transitions** — A song ending on
  its own, a headset media button skip, a shuffle pick, or a repeat-one
  restart all now animate the album-art strip exactly as a transport
  button press does. Previously these transitions arrived as
  instantaneous snaps: the audio changed, the strip's slots updated,
  but nothing moved. The animation is now the default, and only a
  re-publication of the same window — a pause, a resume, a seek, a
  metadata poke — leaves the strip still.
- **Repeat-one has its own animation** — Under repeat-one, a next or
  previous press restarts the current song rather than advancing. The
  strip's response matches that behaviour: instead of the full commit
  slide a normal advance produces, it runs a below-threshold nudge — a
  small glide toward the neighbour position and back to rest. The
  animation reads as an acknowledgement that the song restarted, not
  as the beginning of an advance. The same animation runs when the
  track loops on its own, so a deliberate press and an involuntary
  restart are visually identical.
- **Rejected button presses glide back** — Previously, under
  repeat-one, a next or previous press started the full commit slide
  toward the neighbour and then jumped straight back to rest when the
  service refused to advance. The return is now a glide using the same
  method a below-threshold swipe uses, so the press and the release of
  a committed swipe under the same mode produce the same visible shape.
- **Sort reveals a settled list** — The sort overlay no longer hides
  on a timer measured from the sort's start. It hides only after the
  RecyclerView has laid out and drawn the frame containing the sorted
  list, so the moment the overlay disappears is the moment the user
  sees the final ordering. Nothing repaints after the overlay's exit.
- **Sort no longer stalls the spinner** — The playlist reorder ran in
  two places on the main thread. The service's sort-mode receiver
  sorted the authoritative playlist inline in its broadcast callback,
  and the playlist adapter ran `DiffUtil` on the fully reordered list
  when the activity handed it back. Both are now off the main thread.
  The spinner rotates freely for the entire sort window.

### Fixed

- **Shuffle auto-advance disagreed with the button and the window** —
  Under shuffle with repeat off, the automatic advance at the end of a
  track used the positional next index, while the transport button and
  the carousel's window both used the shuffle-chosen pick. The
  carousel therefore snapped without animating on every natural track
  end under that mode combination, because the incoming current path
  matched neither of the previous window's neighbours. The
  auto-advance now uses the same picker the button and the window use,
  so all three agree on the destination.
- **Repeat-one auto-loop did not return** — Under repeat-one, when a
  track looped on its own, the strip used to slide the neighbour fully
  into the viewport and then cut back to the current song. The visual
  suggested an advance had happened and been reversed. The animation
  is now the below-threshold nudge, matching what a press under the
  same mode produces.
- **Repeat-one next/prev button had no matching animation** — The
  window used to publish null neighbours under repeat-one, so the
  carousel had nowhere to slide. A next or previous press showed no
  animation while the audio restarted. The window now publishes the
  positional neighbours — the songs a next or previous request would
  advance to if repeat-one were off — and the carousel slides toward
  one of them before returning.

### Under the Hood

- `PlaybackWindow` gained a `generation` field: the service's
  monotonic transition counter at the moment the window was produced.
  Equality and hash code now include it, so two windows with the same
  three paths but different generations are distinct values. A
  re-publication (a pause, a resume, a seek, a metadata poke) carries
  the same generation; a fresh transition (a prepare for a new song, a
  reprepare that restarts the current song, a transition to idle)
  advances it. The carousel uses this to tell a restart apart from a
  re-publication even though the three paths are unchanged in both
  cases.
- `MusicService.peekNextIndex(int)` and `peekPreviousIndex(int)` no
  longer special-case repeat-one. They return the positional or
  shuffle-chosen neighbour under every repeat mode. The window
  therefore always has somewhere for the carousel to slide toward.
- `MusicService.getWindowSnapshot()` publishes the peek result without
  nulling the neighbours for repeat-one, and carries `mGeneration` on
  the returned window.
- `MusicService.playNextSong()` and `playPreviousSong()` restart the
  current song under repeat-one by setting `mTargetIndex = baseIndex`
  and `mForceReprepare = true`. The reprepare is what makes
  `reconcile()` begin a fresh transition even though the target index
  has not changed. The three-way switch semantics — off, all, one —
  are now uniform across every entry point: transport buttons, headset
  media buttons, notification actions, and the automatic advance at
  the end of a track.
- `MusicService.onCompletion()` uses `computeNext` whenever shuffle is
  on, under repeat off or all. Previously the repeat-off shuffle path
  was positional, which made the auto-advance disagree with both the
  window and the button.
- `MusicService` gained a single-thread `mSortExecutor`. The sort-mode
  receiver captures the current playlist, the anchor path, and the
  requested sort mode on the main thread, sorts a copy on the executor,
  and posts the swap and the broadcast back to the main thread. A newer
  sort-mode change supersedes an older one through a comparison of the
  captured mode against the current field; a superseded task exits
  without touching state. The executor is shut down in `onDestroy`.
- `MusicPlayer.mUpdatePlayerReceiver` now passes the broadcast's
  `generation` extra to the `PlaybackWindow` constructor as its fourth
  argument. The generation was already parsed for the activity's own
  gate; the change is one line.
- `MusicPlayer.playNextSong()` and `playPreviousSong()` choose the
  carousel animation from the service's current repeat mode: the
  below-threshold nudge when repeat-one, the full commit slide
  otherwise. The mode check is a single `getRepeatMode() == 2` and does
  not race in practice — repeat mode is a user setting that changes far
  slower than a button press round-trip.
- `AlbumArtCarousel` gained a `SwipeState.RESTARTING` state for the
  repeat-one nudge. Both legs of the animation run inside this state so
  a re-publication during the animation is absorbed without
  interrupting it, and only a newer generation aborts it. The state is
  also entered by the automatic restart when the incoming window
  carries the current path unchanged but an advanced generation.
- `AlbumArtCarousel` gained a `RESTART_NUDGE_FRACTION = 0.25f`
  constant, separate from `COMMIT_THRESHOLD_FRACTION = 0.50f`. The
  nudge is deliberately half the commit threshold's travel: it reads as
  a gentle peek rather than a full attempt. The two constants are
  independent so the commit threshold can be tuned without affecting
  the nudge and vice versa.
- `AlbumArtCarousel.setWindow`'s COMMITTING mismatch branch now calls
  `animateToRest` instead of `snapToRest`. The strip glides back from
  wherever the animation had reached rather than jumping, which is what
  produces the rejected-press glide. The same `animateToRest` is used
  by every other return path — below-threshold swipe spring-back,
  refused committed swipe, refused button press, snapshot timeout, and
  the return leg of the nudge — so the return animation is guaranteed
  identical across all cases.
- `AlbumArtCarousel.adoptInIdle` now takes the previous generation and
  recognises two forms of transition: a song change (the current path
  changed to a neighbour — full commit slide) and a restart (the
  current path is unchanged but the generation advanced —
  below-threshold nudge). The generation is the only signal that
  distinguishes them, because the three paths are identical in both
  cases.
- `AlbumArtCarousel.nudgeToNext()` and `nudgeToPrevious()` are the
  public methods the activity uses to run the repeat-one nudge. They
  have the same idle/viewport/neighbour guards as `animateToNext()` /
  `animateToPrevious()` and are the nudge equivalents of those commit
  methods. The carousel has no knowledge of repeat modes; the caller
  selects the animation.
- `AlbumArtCarousel.calculateSampleSize` parameter names were expanded
  from `reqWidth` / `reqHeight` to `requestedWidth` / `requestedHeight`
  to match the codebase's convention of full names for variables.
- `PlaylistActivity` gained a `mSortGeneration` field advanced by every
  `cycleSortMode` call. The hide callback that runs after the
  RecyclerView has settled checks it against the generation its cycle
  was started with, so a superseded cycle cannot take the overlay down
  on the current cycle's behalf.
- `PlaylistActivity.cycleSortMode()` now updates the adapter and
  performs the focus scroll synchronously in the same main-thread turn,
  via a new `scrollToCurrentSongImmediately()` method. The new order
  and the new scroll position land in the same RecyclerView layout
  pass. The overlay is hidden only after that settled frame has been
  drawn, via a new `hideSortOverlayAfterListSettles()` method that
  registers a `ViewTreeObserver.OnPreDrawListener`, removes it on the
  first callback, and posts the hide from inside it. The
  `SORT_OVERLAY_MIN_DURATION_MS` floor is applied on top of the settle
  wait by `scheduleSortOverlayHideAtMinDuration`.
- `PlaylistAdapter.updateList(List<Song>)` gained an overload taking a
  `reorderOnly` flag. The original signature is preserved and delegates
  with the flag set to `false`. A caller that knows the two lists
  contain the same songs in a different order — the sort cycle is the
  case this was added for — passes `true` and the adapter replaces the
  whole list in one step instead of running `DiffUtil`. The flag is a
  hint, not a contract: a caller that is uncertain can pass `false` and
  let `shouldBulkReplace` decide.
- `PlaylistActivity.cycleSortMode()` calls `updateList` with
  `reorderOnly = true`, opting in to the fast path. Every other
  `updateList` call site in the file is unchanged and keeps the
  `shouldBulkReplace` / `DiffUtil` behaviour for the small updates the
  app already uses it for — genre updates, metadata edits, incremental
  progress additions.

---

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
displayed in its next slot is the song the the service plays. Rapid
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

1. Tap the MANAGER button on the main screen
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