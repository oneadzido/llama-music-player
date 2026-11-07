# Llama Music Player v1.4.1

**Release date:** Nov 7, 2026

---

### Overview

This release closes the playback-scope end-of-song defect, hardens the
playback notification against every known path by which it could be
lost, and corrects two long-standing UI weaknesses in the folder
manager and the playback-scope dialog.

The playback-scope defect was the visible symptom. When the user
changed the playback scope mid-song and the new scope excluded the song
that was playing, the service kept that song alive through a grace
period — the mechanism that lets a favourite toggle silence nothing —
but placed it at the end of the current playlist. Under repeat off,
when the song ended, the completion handler asked the current playlist
"is this the last index?" and, seeing that the grace-period survivor
was indeed last, paused. The new scope's songs were never played.
Under repeat all the same handler wrapped to index 0, which happened
to be a scope song, so the new scope played; and under shuffle the
random pick happened to exclude the grace-period survivor because it
was last, so the new scope played. Only repeat off paused. The fix
makes every non-repeat-one mode ask the scoped playlist — the songs
the user actually asked for — whether there is a next song, and pause
only when the current song is genuinely the last scope song or the
scope is empty.

The notification fixes address the paths by which the playback
notification could disappear. A failed foreground promotion no longer
rolls back the render, so a missing `POST_NOTIFICATIONS` grant on
Android 13+ leaves the notification visible instead of killing the
service. The notification cancel branch is refused while the service
reports a transition pending, so a failure that briefly leaves the
state at `IDLE` before its recovery reconcile cannot cancel and
re-post the notification in a visible flicker. The sticky-restart path
— a service command with a null intent, which is what the system sends
after the process has been reclaimed — now rebuilds the notification
from the persisted state instead of leaving the shade empty until the
user opens the player. A failed render no longer advances the
committed generation, so the next poll retries the render instead of
treating it as already committed. A failed render also rolls the media
session back to the previous snapshot, so the notification and the
media session never disagree about what is playing.

The two UI fixes close weaknesses that were present since the features
landed. The playback-scope dialog's title, like every dialog title in
the app, takes the leftover horizontal space between CANCEL and APPLY;
unlike every other dialog title in the app, it did not truncate with
an ellipsis and did not carry start/end padding. A translation longer
than the available width ran under the CANCEL button or the APPLY
button. The folder manager's SCOPE button did not carry the debounce
every other button in the app carries, so a rapid double-tap could
open two `PlaybackScopeDialog` instances, the second on top of the
first. Both are corrected here.

---

### Fixed

- **Playback scope: repeat off pauses after a scope change that
  excludes the playing song.** The completion handler now asks the
  scoped playlist, not the grace-period-extended current playlist,
  whether there is a next song. When the current song is not in the
  new scope, the completion handler advances to the scope's first
  song. When the current song is in the new scope and is not the last
  song, it advances to the next scope song. When the current song is
  the last scope song, or the scope is empty, it pauses at the end of
  the queue. Repeat all and shuffle are unaffected in behaviour but
  now read from the scoped playlist directly rather than relying on
  the grace-period append happening to sit at the last index.

- **Playback scope: shuffle pauses when the scope is empty.** A
  scope change to an empty scope under shuffle, with a
  grace-period survivor, previously looped that survivor
  indefinitely: the completion handler's shuffle branch called
  `pickShuffleNextInScope(fromIndex)`, and that helper's
  single-song fallback resolved the lone scope song back to the
  current song, so the survivor restarted forever. The completion
  handler now pauses in every non-repeat-one mode when the scoped
  playlist is empty, before any shuffle pick is attempted.
  `peekShuffleNextIndex(fromIndex)` itself was not changed — its
  `playlistSize <= 1` early return still returns `fromIndex`, which
  is correct for a playlist the carousel is asking about, and the
  completion handler's new `scopeEmpty` check runs before the
  shuffle branch reaches that code.

- **Notification lost on a failed foreground promotion.**
  `performPlaybackUpdate()` posts the notification with `notify()`
  first and attempts the foreground promotion separately. A
  `SecurityException` from the promotion — the missing
  `POST_NOTIFICATIONS` grant on Android 13+ — no longer rolls back
  the render, so the notification remains visible in the shade and
  the next sync retries the promotion. Before the fix, the rollback
  left the notification invisible and the service died at the
  foreground-service timeout.

  The notification remains visible when `notify()` itself
  succeeds and the promotion fails. When `notify()` throws a
  `SecurityException` — the permission missing before the post
  rather than before the promotion — the render reports failure
  and the state rolls back, so the notification is not posted.
  The split is: post with `notify()`, then attempt the
  promotion, and treat the two failures separately.

- **Notification flickers during a prepare failure's IDLE window.**
  `onPrepareFailed` and `onPlayerError` both set
  `mDeferredReconcile` before posting the recovery reconcile, so
  the service's new `isTransitionPending()` reports true through
  the window between the failure and the reconcile. The two
  methods set the flag differently: `onPrepareFailed` sets it only
  on its fallback path, when no user request was already
  deferred; `onPlayerError` sets it unconditionally on every path.
  Both reach the same effect — the notification's cancel branch,
  which is refused while a transition is pending, cannot cancel
  and re-post the notification through the window.

- **Notification gone after a sticky restart.** `onStartCommand` with
  a null intent now calls `restoreNotificationFromPersistedState()`,
  which reads the persisted last song path, resolves the song, and
  posts a paused snapshot through `commitFromBroadcast`. The shade is
  no longer empty after a process kill until the user opens the
  player.

- **Notification render retries were blocked by a failed render.**
  `commitFromBroadcast` and `syncOnce` now advance
  `mLastCommittedGeneration` only when the render succeeds. A failed
  render at generation `G` leaves the committed generation at `G−1`,
  so the next poll at `G` re-attempts the render instead of treating
  it as already committed.

- **Notification and media session disagree after a failed render.**
  `applySnapshot` now calls `applyToMediaSession` with the previous
  snapshot on failure, so the media session rolls back with the
  notification. Before the fix, the notification rolled back to the
  previous state and the media session stayed at the new state.

- **Library notification could outlive its operation.**
  `PlaylistManager.runGenreLoad` now tears down the library
  notification through
  `NotificationController.dismissLibraryStatus` on both the
  empty-scope and the completion paths, in addition to
  `ToastManager.hidePersistentOverlay`. The library notification is
  the mirror of the toast overlay, and the mirror is now torn down
  through the same owner that created it.

- **Playback-scope dialog title could run under the edge buttons.**
  The dialog's title takes weight 1 in a horizontal layout between
  the CANCEL button at the start and the APPLY button at the end,
  so a translation longer than the available width could extend
  under the CANCEL label or under the APPLY label. Every other
  dialog title in the app truncates with an ellipsis before it
  reaches either button; this one did not. The title now carries
  `setEllipsize(TruncateAt.END)` and 8dp start and end padding
  (`HEADER_COUNT_MARGIN_START_DP` on each side). A long
  translation truncates with an ellipsis and sits 8dp clear of
  both buttons. Before the fix, a German `BEREICH AUSWÄHLEN` or an
  Arabic translation of equivalent length could obscure part of
  the CANCEL or APPLY label.

- **Folder manager's SCOPE button could open two dialogs on a rapid
  double-tap.** The SCOPE button had no debounce, so a tap that
  landed twice within the framework's double-tap window could open
  two `PlaybackScopeDialog` instances, one on top of the other.
  Every other button in the app carries a debounce; this one did
  not. The click handler now records the tap in
  `mLastScopeClickTime` and ignores a second tap that lands within
  `CLICK_DEBOUNCE_DELAY_MS` (500 ms), matching the CANCEL button's
  debounce. A physical double-tap on SCOPE opens one dialog.

---

### Under the Hood

**Playback scope**

- `MusicService.onCompletion()` — rewritten. Reads `mScopedPlaylist`
  directly to decide whether there is a next song. Repeat one restarts
  the current song. Repeat all wraps to the next scope song. Shuffle
  picks from the scope through `pickShuffleNextInScope(int)`. Repeat
  off advances to the next scope song, or pauses at the end of the
  queue. Every non-repeat-one branch pauses when the scope is empty,
  and the check runs before the shuffle or wrap logic on every
  branch.

- `MusicService.pauseAtEndOfQueue(int)` — new helper. Pauses the
  player, rewinds to position 0, releases the wake lock, saves state,
  and broadcasts. Replaces the three inline pause blocks that
  previously appeared in `onCompletion`.

- `MusicService.pickShuffleNextInScope(int)` — new helper. Picks a
  random song from `mScopedPlaylist`, different from the current song
  when the scope holds more than one. Advances the shuffle history
  and clears the shuffle peek memos, matching
  `pickShuffleNext(int)`'s side effects. The pick is always a scope
  song, without relying on the grace-period append happening to sit
  at the last index.

**Notification resilience**

- `MusicService.isTransitionPending()` — new public method.
  Returns true when the state machine is `PREPARING`, when a
  deferred reconcile is queued, when a forced reprepare is
  pending, or when the target index no longer matches the current
  index. Consumed by `NotificationController.syncOnce()`, which
  reaches it through its `Service` reference and an `instanceof`
  `MusicService` guard; the controller's other service calls go
  through the `ServiceStateSource` interface, and
  `isTransitionPending()` is the one method it calls directly on
  the service.

- `MusicService.mStateSource.snapshotForNotification()` — the
  `PREPARING` branch now also covers the `IDLE`-with-pending
  window: when the state is `IDLE` and `isTransitionPending()`
  returns true, the method returns `getCurrentSong()` rather than
  `getDisplaySong()`, so the notification continues to show the
  current-committed song through that window rather than
  advancing to the target and back. The window is a transient
  state between a prepare failure and its recovery reconcile.

- `MusicService.onPrepareFailed(long, int)` — sets
  `mDeferredReconcile` on its fallback path, when no user request
  was already deferred, before posting the recovery reconcile.
  The flag reports the pending transition through the `IDLE`
  window so the notification's cancel branch is refused.

- `MusicService.onPlayerError(int, int)` — sets
  `mDeferredReconcile` unconditionally before posting the
  recovery reconcile. The unconditional set is correct here
  because the error path does not have the user-request-deferred
  case `onPrepareFailed` has: every error leads to the fallback
  advance, and every fallback advance needs the window covered.

- `MusicService.onStartCommand(Intent, int, int)` — a null intent
  or a null action now restores the notification from persisted
  state. The action-dispatch path is unchanged.

- `MusicService.restoreNotificationFromPersistedState()` — new
  helper. Reads the persisted last song path, resolves the song
  against the database, builds a paused `RenderSnapshot`, and
  commits it through `commitFromBroadcast`. No playback starts.

- `NotificationController.syncOnce()` — the cancel branch is
  refused while `MusicService.isTransitionPending()` returns true.
  `mLastCommittedGeneration` is advanced only when the render
  succeeds.

- `NotificationController.applySnapshot(RenderSnapshot)` — on render
  failure, calls `applyToMediaSession` with the previous snapshot so
  the media session rolls back with the notification. The previous
  snapshot is captured before the new one is committed.

- `NotificationController.performPlaybackUpdate()` — posts the
  notification with `notify()` first, then attempts the foreground
  promotion in a separate try/catch. A promotion failure logs and
  leaves `mPlaybackForeground` false so the next sync retries; the
  notification that `notify()` successfully posted stays visible.
  A `notify()` failure of its own — the permission missing before
  the post — returns false and rolls the render back. The two
  failures are handled separately because they leave the shade in
  two different states.

- `NotificationController.commitFromBroadcast(long, RenderSnapshot)`
  — advances `mLastCommittedGeneration` only when the render
  succeeds. When the snapshot equals what is already rendered, the
  generation is still advanced (a re-publication is a successful
  commit even if it is a no-op for the notification).

**Library notification teardown**

- `PlaylistManager.sApplication` — new static field. Set in the
  constructor, cleared in `shutdown()`.

- `PlaylistManager.runGenreLoad(String, String)` — the empty-scope
  and completion paths now call
  `NotificationController.dismissLibraryStatus(operationId)` after
  `ToastManager.hidePersistentOverlay(operationId)`. The mirror is
  torn down through its owner.

**Scope dialog and folder manager**

- `PlaybackScopeDialog.buildTopBar()` — the title text view now
  calls `setEllipsize(android.text.TextUtils.TruncateAt.END)` and
  `setPadding(dpToPx(HEADER_COUNT_MARGIN_START_DP), 0,
  dpToPx(HEADER_COUNT_MARGIN_START_DP), 0)`. The padding value is
  the same 8dp constant the header uses for its count's start
  margin, so the two insets are one value. The ellipsize was the
  pattern every other dialog title in the app already used; the
  playback-scope dialog was the one that did not.

- `FolderManager` gained `mLastScopeClickTime`, initialized to
  `0L`. The SCOPE button's `onClick` now checks the tap against
  `CLICK_DEBOUNCE_DELAY_MS` and updates the field on every
  accepted tap, mirroring the CANCEL button's debounce. Before the
  fix, the handler checked only `isUpdating()` and then opened the
  dialog, so a rapid double-tap opened two dialogs.

- `FolderManager.STATE_SAVE_DELAY_MS` — type changed from `int` to
  `long`, and the literal changed from `150` to `150L`.
  `CLICK_DEBOUNCE_DELAY_MS`'s literal changed from `500` to `500L`.
  The `mLastCancelClickTime` initializer changed from `0` to `0L`.
  The two `postDelayed(..., 100)` calls in `onResume` and
  `onRestart` changed to `postDelayed(..., 100L)`. No behaviour
  change; the values are the same.

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

1. Download the APK file.
2. Enable "Unknown Sources" in your device settings.
3. Open the APK file and tap "Install".

---

## First Time Setup

1. Tap the MANAGER button on the main screen.
2. Tap IMPORT FOLDER to select your music folder.
3. Tap UPDATE to save your selection.
4. The app will scan your folder and build the playlist.

---

## How to Use

| Feature | How to Access |
|---------|---------------|
| Favorites | Double-tap the album art, tap the heart on the player's screen, or tap a playlist row's heart |
| Playback scope | Tap MANAGER, then SCOPE in the folder manager header |
| Exit button | Tap the exit button on the main screen twice within one second |
| Equalizer | Tap EQUALIZER on the main screen |
| Metadata Editor | Tap METADATA on the main screen |
| Folder Manager | Tap MANAGER on the main screen |
| Playlist | Tap PLAYLIST on the main screen |
| Themes | Tap LLAMA on the main screen, then THEMES |
| Language | Tap LLAMA on the main screen, then the language button in the version header |
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

Licensed under the MIT License.

Copyright (c) 2026 Richard Korbla Adzido.

Made with love in Ghana.