# Llama Music Player v1.3.8

**Release date:** Oct 27, 2026

---

### Overview

Five changes ship in this release. The notification surface commits the
same state the player screen commits, on the same main-thread turn. The
notification's rendered content does not advance during a prepare — it
stays on the song the player screen is still showing until the
transition commits. The notification's transport buttons dispatch
directly to the service rather than through the media session's
broadcast receiver. The resume path commits PLAYING synchronously so
every surface flips on the same turn as the user's tap. And the service
no longer pre-commits or re-posts the notification around a transport
action, so a tap produces exactly one commit and the icon never flashes
the pre-tap state.

`NotificationController.commitFromBroadcast` is the notification's
primary commit path. `MusicService.broadcastUpdate` calls it
immediately after `sendBroadcast` returns, with the exact snapshot the
broadcast described. The player screen commits inside its
`UPDATE_PLAYER` receiver, which runs synchronously inside that same
`sendBroadcast` call. Both commits land on the same main-thread turn
with the same snapshot: same title, artist, art, playing flag,
generation.

`broadcastUpdate` is the only path that commits the notification's
rendered content. `onStartCommand` does not commit the notification
before processing a transport action — that pre-action commit showed
the state the user had already left, and on some devices the two posts
interleaved and produced a brief flash of the old icon after the new
one. `onPlaybackStarted` and `onPlaybackResumed` do not re-post after
`broadcastUpdate` — a second `startForegroundSync` after the commit
re-posted the same notification asynchronously, producing the same
flash. A notification transport tap now produces exactly one commit,
fired by `broadcastUpdate` when the state machine reaches the new
state.

During PREPARING, `snapshotForNotification()` returns the
current-committed song — the song the player screen is still
showing — rather than the target. The poll that runs on the
controller's cadence therefore reads the same song the player screen
is showing and cannot advance the notification ahead of the player.
Both surfaces continue to show the song the previous transition
committed until `broadcastUpdate` fires; both advance to the target
on that turn.

The three transport PendingIntents the notification attaches to
play/pause, next, and previous dispatch through
`PendingIntent.getService` with private actions (`ACTION_PLAY_PAUSE`,
`ACTION_NEXT`, `ACTION_PREV`). `MusicService.onStartCommand` handles
each synchronously on the main thread by calling `togglePlayPause()`,
`playNextSong()`, or `playPreviousSong()` directly. This is one hop —
the same path the close button uses — rather than the three-hop
broadcast receiver → service start → media-session dispatch that
`MediaButtonReceiver` and
`MediaSessionCompat.dispatchMediaButtonEvent` introduce. The media
session still receives lockscreen, bluetooth, and headset media
buttons through the unchanged `ACTION_MEDIA_BUTTON` path; only the
notification's own buttons take the direct path.

`doResume` commits PLAYING synchronously, before `player.start()` is
dispatched to the player thread, so the play/pause icon, the
notification title and art, the media session state, and the player
screen's button all flip on the same main-thread turn as the user's
tap. The async continuation exists only to roll the state back to IDLE
if `player.start()` actually fails, and the rollback is guarded by
both `mStartToken` and `mState` so a subsequent user action is not
clobbered by a start that began before it. `doPause` bumps
`mStartToken` at entry so a pause that arrives during an in-flight
resume invalidates the resume's failure continuation. The `READY`
branch of `togglePlayPause` routes through `doResume`, so a cold-start
play uses the same synchronous-commit pattern as a `PAUSED` resume.

A generation filter drops any notification read whose generation is
strictly lower than the last committed generation, so a poll that
captured state before a broadcast cannot re-apply it after the
broadcast lands.

---

### Fixed

- **Notification icon does not flash the pre-tap state** — The
  service no longer commits the notification before processing a
  transport action, and no longer re-posts the notification after
  `broadcastUpdate`. A pause tap produces exactly one commit — the
  one `broadcastUpdate` fires when `doPause` has committed `PAUSED` —
  and a resume tap produces exactly one commit — the one
  `broadcastUpdate` fires when `onPlaybackResumed` has committed
  `PLAYING`. The icon flips once and stays. On the previous code path
  the pre-action commit showed the pre-tap state, and the post-commit
  `startForegroundSync` re-posted the same notification asynchronously;
  on some devices the two posts interleaved and the user saw a brief
  flash of the pre-tap icon after the post-tap one.

- **Notification never advances ahead of the player screen** —
  During PREPARING, `snapshotForNotification()` returns the
  current-committed song, not the target. The poll that runs on the
  controller's cadence reads the same song the player screen is
  showing and therefore cannot advance the notification ahead of the
  player. `broadcastUpdate` fires when the transition commits, and
  both surfaces advance on that turn. Without this, the poll would
  advance the notification to the target while the player screen
  still shows the outgoing song, and the two surfaces would disagree
  for the duration of every prepare.

- **Notification play/pause button fires play/pause** — A tap on the
  notification's play/pause action dispatches the play/pause action
  to `MusicService.onStartCommand`, which handles it synchronously on
  the main thread. The three transport PendingIntents carry distinct
  request codes and private actions that are not shared with the
  media session. The notification button reaches the state machine
  in one hop — the same path the close button uses — rather than the
  three-hop broadcast receiver → service start → media-session
  dispatch that routed all three buttons through the same dispatch
  point.

- **Notification play/pause icon matches the audio on every
  transport action** — The icon reads the destination of the current
  transition from the state machine, the same value the player
  screen's icon reads, and commits it on the same main-thread turn as
  the player screen. A play/pause tap from the notification, the
  lockscreen, a headset, or the player screen produces the same icon
  change on both surfaces on the same frame.

- **Rapid transport taps are handled correctly** — The resume path
  commits PLAYING synchronously before dispatching `player.start()`,
  so a pause that follows a resume within the transport debounce
  window sees `PLAYING` and is honored. `doPause` bumps
  `mStartToken` to invalidate an in-flight resume's failure
  continuation. A next or previous that arrives during a prepare is
  stored as the deferred target and played once the current prepare
  completes. Every sequence of rapid taps reaches the state machine,
  and the state machine handles it correctly.

- **Notification title, artist, and art match the player screen
  through every transition** — When a transition begins, neither
  surface advances. Both continue to show the song the previous
  transition committed. When the transition commits, both advance to
  the target on the same frame.

- **Foreground promotion is retried on failure** — A
  `startForeground` call that throws leaves the controller's
  foreground flag `false`, so the next sync retries the promotion.
  The flag is set only after the call succeeds.

---

### Under the Hood

- `NotificationController.ServiceStateSource` — `long
  getGeneration()`. Returns the service's monotonic transition
  counter. Both `commitFromBroadcast` and `syncOnce` use it to drop
  stale reads.

- `NotificationController.commitFromBroadcast(long, RenderSnapshot)`
  — the notification's primary commit path.
  `MusicService.broadcastUpdate` calls it after `sendBroadcast`
  returns, with the exact snapshot the broadcast described. Marshals
  to the main thread from any other thread. Drops any call whose
  generation is strictly lower than the last committed generation.

- `NotificationController.syncOnce()` — reads
  `ServiceStateSource.getGeneration()` first. When the generation is
  strictly lower than `mLastCommittedGeneration`, the read is
  dropped. When it is at or beyond that generation, the snapshot is
  committed and `mLastCommittedGeneration` advances.

- `NotificationController.mLastCommittedGeneration` — the generation
  as of the last committed snapshot. Advanced by both commit paths.
  Reset in `detachService()`, `clearPlaybackNotification()`, and
  `shutdown()`.

- `NotificationController.performPlaybackUpdate()` — the
  `mPlaybackForeground` flag is set only after
  `Service.startForeground` returns.

- `NotificationController.startForegroundSync()` — removed. It had no
  remaining callers after the pre-action commit and the post-commit
  re-post were both removed from `MusicService`. The foreground
  promotion now happens exclusively inside `performPlaybackUpdate`,
  driven by `commitFromBroadcast` — the only path that commits the
  notification.

- `NotificationController` transport PendingIntents — the three
  intents the notification attaches to play/pause, next, and
  previous are built through a
  `buildTransportPendingIntent(int requestCode, String action)`
  helper with distinct request codes (1, 2, 3) and actions
  `ACTION_PLAY_PAUSE`, `ACTION_NEXT`, `ACTION_PREV`.
  `MusicService.onStartCommand` handles each action synchronously on
  the main thread. This is one hop — the same path the close button
  uses — rather than the three-hop broadcast receiver → service
  start → media-session dispatch. The media session still receives
  lockscreen, bluetooth, and headset media buttons through the
  unchanged `ACTION_MEDIA_BUTTON` path; only the notification's own
  buttons take the direct path.

- `MusicService.mStateSource` — implements `getGeneration()`,
  returning `mGeneration`.

- `MusicService.mStateSource.snapshotForNotification()` — returns the
  current-committed song's snapshot during PREPARING, not the
  target. The player screen does not advance to the target until the
  transition commits and fires `broadcastUpdate`, and the
  notification must match the player screen. Without this, the poll
  would advance the notification to the target while the player
  screen still shows the outgoing song.

- `MusicService.broadcastUpdate()` — after `sendBroadcast(intent)`,
  builds the same snapshot from the same main-thread state and calls
  `mNotificationController.commitFromBroadcast(generation, snapshot)`.
  The generation is captured before `sendBroadcast` so the commit
  and the broadcast carry the same value. This is the only path that
  commits the notification.

- `MusicService.onStartCommand()` — the pre-action
  `startForegroundSync()` block is removed. Every action commits
  through its own path (`doPause`, `doResume`, or the completion
  callback that fires `broadcastUpdate`). Committing before the
  action showed the pre-action state.

- `MusicService.beginTransition(int, int)` — does not commit the
  notification. During PREPARING the notification continues to show
  the previous song; it advances to the target when the transition
  commits through `broadcastUpdate`.

- `MusicService.doResume()` — commits PLAYING synchronously through
  `onPlaybackResumed()` before dispatching `player.start()` to the
  player thread. The async continuation exists only for the failure
  rollback, which is guarded by both `mStartToken` and
  `mState == ServiceState.PLAYING`. On failure, the rollback sets
  the state to IDLE, releases the player, releases the wake lock,
  persists the rolled-back state, broadcasts, and re-enters
  reconciliation.

- `MusicService.doPause()` — bumps `mStartToken` at entry. The bump
  invalidates any in-flight resume continuation, so a pause that
  arrives while a resume's `player.start()` is still on the player
  thread is not followed by a rollback that clobbers the committed
  `PAUSED`.

- `MusicService.togglePlayPause()` — the `READY` branch routes
  through `doResume()` so a cold-start play commits PLAYING
  synchronously, matching the `PAUSED` resume path.

- `MusicService.onPlaybackResumed()` — called synchronously from
  `doResume()` before the async `player.start()`. Commits the
  PLAYING state, the wake lock, the media session activation, the
  persisted state, and the notification through `broadcastUpdate`.
  No separate `startForegroundSync` call after the commit;
  `broadcastUpdate`'s commit already promotes or demotes based on
  the state.

- `MusicService.onPlaybackStarted()` — commits the transition
  through `broadcastUpdate`. No separate `startForegroundSync` call
  after the commit, same reason.

- `MusicService.handleMediaButton(int)` — the play/pause, next,
  previous, and bluetooth volume keys are debounced at the same
  interval as the notification's transport actions and the media
  session's callbacks. A rapid double-press on a headset collapses
  to one action, matching the debounce every other transport surface
  applies.

- `MusicService.prefetchNotificationArt()` — the art-resolve poke is
  guarded by `mState != ServiceState.PREPARING`. An art resolve that
  lands during PREPARING is picked up by the commit that fires from
  `broadcastUpdate`; an art resolve that lands after the commit pokes
  the controller on the same main-thread turn as the resolve.

- `MusicService` javadoc — "The Commit Turn" describes
  `broadcastUpdate` as the only notification commit path and
  describes the PREPARING hold. "Play State Truth" describes the
  synchronous PLAYING commit at resume.

- `NotificationController` javadoc — "Commit Model", "Generation
  Filtering", and "Transport PendingIntents" sections document the
  synchronous commit path, the generation filter, and the direct
  transport dispatch.

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