# Llama Music Player v1.2.0

**Release date:** Sep 30, 2026

## Overview
Two new interaction features, a split-screen layout that reshapes the
whole app, a redesigned loading indicator, a process-wide overlay
architecture, and a three-slot album-art carousel underneath it all.

## What's New

- **Volume-aware playback** — Reaching volume 0 pauses playback.
  Raising the volume above 0 resumes it. A pause you made yourself,
  or one caused by a Bluetooth disconnect or an audio-focus loss, is
  never undone by raising the volume.
- **Album-art volume gesture** — Swipe up or down along the left or
  right edge of the album art to change the volume. A normal toast
  shows the current level as a percentage.
- **Split-screen layout** — Every screen now scales its vertical
  margins and paddings with the window height. The album art
  resizes with the window: at full height it is shown at 300dp; in
  a tall split window it shrinks to 180dp and the volume gesture
  remains available; in a short split window the art hides so the
  title, artist, progress bar, timer, and controls take the space.
  The equalizer, folder manager, metadata editor, and playlist
  screens compress their own vertical spacing on the same rule, so
  none of them scrolls more than it has to in a short window.
- **Redesigned loading indicator** — A rotating ring with a bright
  arc travelling around a dim track, in place of the previous static
  circle. The arc's position is legible on every frame, so the
  indicator reads as moving regardless of how long the underlying
  work takes.

## Album-art carousel

The album art is now a three-slot strip: the previous song's art on
the left, the current song's art in the middle, the next song's art
on the right. Swipe horizontally across the art and the strip follows
your finger. Release past the midpoint and the strip commits to the
neighbouring song; release before it and the strip springs back. The
carousel owns the decode, the cache, and the neighbour loading, so a
swipe reveals the next song's art without waiting for it to decode.
Rapid skip-forward and skip-back from the carousel itself are
handled by a generation counter that discards any decode whose window
has already moved on.

## Media carousel

The notification and the system media controls follow every playback
transition, including rapid skip-forward and skip-back from the
carousel itself. The metadata is delivered to the notification and
the media session as it arrives, with the themed placeholder as the
large icon while a song's album art is still decoding, and with the
real bitmap once the decode resolves. The playback state is written
on every awareness tick, so a state change the carousel missed on
one tick is retried on the next.

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

Cold start, the metadata editor's reads, and the folder-update cycle
all show the same dim-and-spinner overlay. The overlay is
process-wide and reason-keyed: the spinner and the dim background
belong to the process, not to any single activity, and the overlay
follows the user across navigation. When you back-swipe out of the
folder manager while a scan is running, the overlay is still there
when you land in the player. The playlist rebuild now uses the in-app
overlay instead of the previous background notification, so the
transition from folder scan to genre loading stays inside the app's
own surface. The spinner's rotation is bound to its attach state,
so the animation runs exactly while the view is on screen.

## Under the Hood

- `OverlayManager` — the process-wide, reason-keyed dim-and-spinner
  overlay. A singleton that registers itself as an
  `Application.ActivityLifecycleCallbacks` listener and reattaches
  the overlay to the current activity on every resume. The overlay
  is visible while at least one reason token is held, so the
  cold-start library load, the metadata editor's reads, and the
  folder-update cycle can all hold the overlay without interfering
  with each other. The spinner is a private view with a
  lifecycle-bound rotation animator, started on attach and cancelled
  on detach. A periodic health check releases any reason held longer
  than the stuck timeout.
- `AlbumArtCarousel` — the three-slot album-art strip. Owns the
  strip, the three `ImageView`s, the rolling window of decoded
  bitmaps, and the async loading of the previous and next neighbours.
  The shared `LruCache` and `ExecutorService` are supplied by
  `MusicPlayer`, so cached bitmaps are reused across the carousel
  and the cold-start preload. A generation counter discards any
  decode whose window has advanced. The current-art fade is
  suppressed when the change came from a swipe, because the carousel
  has already shown the incoming art during the drag.
- `SplitScreenHelper` — captures the original vertical margins and
  paddings of every view in an activity's hierarchy once, and
  reapplies them at a factor derived from
  `Configuration.screenHeightDp`. Four bands: 1.0 at 600dp and
  above, 0.75 at 480dp, 0.5 at 400dp, and 0.25 below. Each activity
  owns one instance; the helper writes from the recorded originals
  on every call, so a return to full height restores the XML
  values exactly. `MusicPlayer` applies two additional rules on top
  of the helper's scaling: the album art's size and visibility, and
  the title and artist combo's symmetric margins.
- `MusicService` monitors the music stream through a receiver for
  `AudioManager.VOLUME_CHANGED_ACTION`, filtered to the music
  stream. The volume-zero marker is set only when the receiver
  itself paused or deferred a pause; every other pause path clears
  it. Only a pause with the marker set is undone by raising the
  volume.
- `MusicService` checkpoints the persisted position on a fixed
  interval inside the existing health monitor, and on
  `onTrimMemory`. The state-change saves remain in place for pause,
  seek, and stop.
- `MusicService.setPlaylist` prepares the anchor paused on every
  cold start. The rule applies whether the anchor came from a
  persisted last-song path or from the fallback to index zero on an
  empty playlist history.
- `MusicService.mStateSource.snapshotForNotification` reports the
  target song's identity during `PREPARING`, so the notification and
  the media carousel advance with the user's tap rather than waiting
  for the transition to complete.
- `NotificationController.applySnapshot` commits a song's identity as
  it arrives, with the themed placeholder as the large icon when art
  has not yet resolved and the real bitmap when it has.
- `MusicPlayer.onConfigurationChanged` reapplies the split-screen
  layout when the window is resized. The album-art size and
  visibility are a pure function of `Configuration.screenHeightDp`.
- `MusicPlayer.proceedWithInitialization` is linear: preload the
  previous song for display, then start and bind the service. The
  activity's preload paints the title, artist, and art while the
  playlist loads, and does not decide whether anything is restored.
- `MusicPlayer`'s album-art container owns every gesture that
  operates on the album art. A single touch listener classifies each
  gesture on the first significant move: a horizontal swipe becomes
  a carousel swipe, a vertical swipe that starts in the left or
  right 20% of the container becomes a volume change, and a
  vertical swipe that starts at the centre is cancelled so it
  reaches whatever sits behind the album art.
- Every activity that displays the overlay or the split-screen
  layout declares the same `android:configChanges` set
  (`keyboardHidden|orientation|screenSize|smallestScreenSize|screenLayout`)
  so a resize runs `onConfigurationChanged` instead of recreating
  the activity.

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
| Previous / next song | Swipe left or right across the album art |

---

## Credits

- jaudiotagger by jthink Ltd
- FontAwesome by Fonticons, Inc.

---

## License

Licensed under the MIT License

Copyright (c) 2026 Richard Korbla Adzido

Made with love in Ghana