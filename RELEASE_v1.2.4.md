# Llama Music Player v1.2.4

**Release date:** Oct 6, 2026

## Overview

Two fixes ship in this release: one in the player's split-screen
layout, one in the incoming-file pathway.

In split-screen mode, when the album art hides to make room for the
title, artist, seek bar, and controls, the title and artist group now
moves up to occupy the space the album art vacated. Previously a
padding band survived above the group after the album art was hidden,
and the artist line was pushed past the visible portion of the window.
The wrapper is now hidden together with the album-art viewport, and
the group is pinned to the top of its slot at the album-art top's
Y offset.

When a user opens an audio file from another app, the file is added
as a loose track, the playlist rebuilds from the database, and the
song plays through the same transition path a transport request
uses. Previously the loose-track insertion announced the change with
a broadcast no component listened for, so the service's playlist was
never rebuilt and the song appeared in the folder list but not in the
playlist. The insertion now announces the change with
`UPDATE_PLAYLIST`, the vocabulary every actor in the process already
listens for.

## Fixed

- **Title and artist fill the space the album art vacates in
  split-screen** — When the window dropped below the minimum height
  that supports the album art, the album-art viewport was set to
  `GONE`, but the wrapper around it kept contributing its top and
  bottom padding. That padding survived as a visible band above the
  song-info group, and the group's centre-gravity layout pushed the
  artist line past the bottom of the visible window. The wrapper is
  now hidden together with the viewport, so its padding is not laid
  out. The title-and-artist group is pinned to the top of its weight
  slot and given a top offset equal to the wrapper's declared top
  padding — the same offset the album-art top carries in full-screen
  mode — so the title lands where the album art used to begin. The
  artist line is now visible. Every view below the group — seek bar,
  transport controls, metadata row — keeps its intrinsic position
  relative to the bottom of the window, because the group's weight
  slot absorbs the difference. Full-screen and shrunk-album-art
  layouts are unchanged: the wrapper is restored, and the group
  returns to its layout-declared gravity and padding.

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

## Under the Hood

- `res/layout/activity_music_player.xml` gained an `albumArtWrapper`
  identifier on the album-art padding container and a `songInfoGroup`
  identifier on the title-and-artist group. Nothing else in the
  layout moved.

- `MusicPlayer.initializeViews()` resolves the two new views and
  captures the group's layout-declared gravity and top padding.
  `MusicPlayer.applyAlbumArtSize()` calls a new
  `applySongInfoLayout(boolean)` after the album-art viewport's size
  and visibility have been applied. The new method hides or restores
  the wrapper, and pins or restores the group's gravity and top
  padding. The wrapper's declared top padding is read from the layout
  rather than duplicated as a constant, so a change to the wrapper's
  padding propagates to the split-screen position without a matching
  edit in the activity.

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