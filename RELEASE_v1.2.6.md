# Llama Music Player v1.2.6

**Release date:** Oct 8, 2026

## Overview

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

## Fixed

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

## Under the Hood

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