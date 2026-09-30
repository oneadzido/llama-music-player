# Llama Music Player v1.2.0

**Release date:** Sep 30, 2026

## Overview
Two new interaction features, a split-screen layout that reshapes the
whole app, a redesigned loading indicator, a process-wide overlay
architecture, and a three-slot album-art carousel that renders the
playback window the service publishes.

## What's New

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

## Album-art carousel

The album art is a three-slot strip: the previous song's art on the
left, the current song's art in the middle, the next song's art on
the right. Swipe horizontally across the art and the strip follows
your finger. Release past the midpoint and the strip commits to the
neighbouring song; release before it and the strip springs back.

The carousel renders a *window* the service publishes — the previous
neighbour's path, the current song's path, the next neighbour's path,
computed by `MusicService.getWindowSnapshot()`. The window is the same
triple the service's own Next and Previous requests commit from, so
the carousel and the service cannot disagree about which song will
play next, which song is the previous, or which song is current. A
window is a value; adopting it is a single method, `setWindow`, and
every path through the service — a transport button, a headset key, a
notification action, a Bluetooth resume, a shuffle re-pick, a sort
reorder, a library reset — publishes a window.

Pressing **Next** or **Previous** animates the same strip. The slide
and the audio transition run in parallel: the button dispatches the
service request immediately, and the carousel slides while the service
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

## Playlist edge fast-scroll

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

## Media carousel

The notification and the system media controls follow every playback
transition, including rapid skip-forward and skip-back from the
carousel itself. The metadata is delivered to the notification and
the media session as it arrives, with the themed placeholder as the
large icon while a song's album art is still decoding, and with the
real bitmap once the decode resolves. The playback state is written
on every awareness tick, so a state change the carousel missed on
one tick is retried on the next.

The notification, the media session, the player UI, and the album-art
carousel all read their display identity from one rule on the service:
during a transition the display song is the target; otherwise it is
the current song. One expression of the rule, four consumers, no
drift.

## Playback restoration

The last song and its playback position restore reliably on every
cold start, including after a force-kill or an OS-initiated process
teardown. The position is checkpointed on a fixed interval while
music is playing, alongside the existing saves at play, pause, and
seek. A force-kill loses at most five seconds of listening position
instead of the whole song.

## Folder import

Importing a folder after a library wipe prepares the first song
paused. The service has no prior playback signal to honour on a cold
start, so it waits for you to press play rather than starting
playback on its own. Restoring a session after a restart is
unaffected: if you were playing when the process went away,
playback resumes at the saved position.

## Loading overlay

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

## Under the Hood

- `PlaybackWindow` — an immutable value type holding the three paths
  the carousel renders: the previous neighbour, the current song, the
  next neighbour. Equality compares the three paths, so republishing
  the same window is a no-op for every consumer. The service is the
  sole producer; the carousel is the sole consumer.
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
- `NotificationController.applySnapshot` commits a song's identity as
  it arrives, with the themed placeholder as the large icon when art
  has not yet resolved and the real bitmap when it has. The
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
  drive the carousel's slide before dispatching the service
  request. When the carousel cannot animate — a commit is already
  in flight, the viewport is not laid out, or the window advertises
  no neighbour in that direction — the call is a no-op and the
  service request still proceeds; the resulting window is adopted
  without animation.

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

1. Tap the MANAGER button in the top right corner
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