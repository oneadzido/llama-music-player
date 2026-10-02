# Llama Music Player v1.2.2

**Release date:** Oct 2, 2026

## Overview

The album-art volume gesture now starts playback from the moment
a song loads, not only after the first tap on the play button. The folder
list scrolls naturally in a short split-screen window instead of being
hidden. Two click-debounce timers that were still reading wall-clock
time have been moved to the monotonic clock the rest of the app uses.

## Fixed

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

## Under the Hood

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