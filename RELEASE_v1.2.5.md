# Llama Music Player v1.2.5

**Release date:** Oct 7, 2026

## Overview

Four changes ship in this release — a Material Design 3 motion system
for the album-art carousel, a unified commit turn, a single-song
playlist resolution for the transport buttons, and a preservation
pass over the incoming-file pathway.

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
target did not change, and the state machine did not transition.

Two rapid incoming files now resolve in order even when their inserts
overlap, and a request that arrives during an in-flight transition is
no longer lost to a failure-recovery heuristic.

## Fixed

- **The album art and the text land on the same frame** — The
  `UPDATE_PLAYER` receiver now builds one commit runnable and
  dispatches it through the carousel's commit contract, so the
  notification, the media session, the seek bar, the title, the
  artist, and the transport glyph all reach their surfaces on the
  same frame the album art settles on.

- **The next and previous buttons no longer schedule a delayed
  glyph update** — The `postDelayed(updatePlayPauseButton, 100)`
  calls that landed a layout pass on a mid-slide frame are gone.
  The glyph is delivered by the receiver through the carousel's
  settle hook.

- **The album-art carousel uses Material Design 3 motion** — The
  commit slide runs on the emphasized decelerate curve. The reveal
  is a staggered cross-fade with an accelerate outgoing leg and a
  decelerate incoming leg. The nudge is driven by spring physics
  with a low-bouncy damping ratio. The reveal's overlay is
  composited on a hardware layer for the duration of the fade.

- **Single-song playlist transport buttons restart the song** — On
  a playlist with one song, the next and previous buttons did
  nothing under repeat-off and repeat-all. They now restart the
  song on every repeat mode. The service's two new methods,
  `willNextRestartCurrent()` and `willPreviousRestartCurrent()`,
  tell the activity whether the request will restart the current
  song or advance to a different one, and the carousel's nudge now
  accepts a neighbour whose path equals the current path so the
  restart has somewhere to slide.

- **A request that arrives during an in-flight transition is not
  lost** — A new `mDeferredReconcile` flag on the service preserves
  the newest target through a prepare failure. Previously the only
  re-entry point was a comparison of the target index against the
  current index in the prepare callbacks, which does not fire when
  a failure-recovery heuristic has already reset the target to a
  positional successor.

- **A rebuild during an in-flight transition no longer redirects
  the target** — When the state is PREPARING and a playlist
  rebuild runs, the rebuild now re-resolves the target's path
  against the new playlist instead of resetting the target from
  the anchor. The transition in flight is preserved.

- **Two rapid loose-track opens resolve in order** — The pending
  slot is consumed only when it was resolved by its own path in
  the rebuilt playlist. The insert's completion no longer re-fires
  the hand-off, so a newer request is not overwritten by an older
  one.

- **The notification surface no longer drifts from the media
  session** — The rendered-state advance is gated on the
  notification post actually landing. Art retention across a
  re-render of the same song is gated on the decode still being in
  flight. A theme change re-applies the session metadata from the
  last committed snapshot.

## Under the Hood

- `AlbumArtCarousel` gained the Material Design 3 motion constants
  (`EMPHASIZED_DECELERATE`, `EMPHASIZED_ACCELERATE`,
  `NUDGE_SPRING_DAMPING_RATIO`, `NUDGE_SPRING_STIFFNESS`,
  `COMMIT_DURATION_MS = 500`, `RETURN_DURATION_MS = 350`,
  `REVEAL_DURATION_MS = 600`,
  `REVEAL_OUTGOING_FRACTION = 0.45`,
  `REVEAL_INCOMING_START_SCALE = 0.96`,
  `REVEAL_OVERLAP_FRACTION = 0.15`). `setWindow` and `adoptInIdle`
  return a boolean — the commit contract. `isAnimating()` reports
  true while `mReturningToRest` is set.

- `MusicPlayer`. The `UPDATE_PLAYER` receiver builds one commit
  runnable. The playlist-update receivers no longer call
  `refreshCarouselFromService`. The `postDelayed(updatePlayPauseButton,
  100)` calls and the `NEXT_PREV_UPDATE_DELAY_MS` constant were
  removed.

- `MusicService` gained `mDeferredReconcile`,
  `willNextRestartCurrent()`, and `willPreviousRestartCurrent()`.
  `setPlaylist` preserves the target through a PREPARING rebuild and
  returns without falling back to index 0 when the pending path is
  not yet findable. `broadcastUpdate` pokes the notification
  controller after dispatch.

- `PlaybackWindow`. The Null Semantics paragraph was corrected to
  describe what the service actually produces.

- `NotificationController` gained `commitNow()`. `syncOnce`
  advances `mLastRenderedSnapshot` only when the post landed.
  `applySnapshot` and `performPlaybackUpdate` return a boolean.

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