# Llama Music Player v1.1.9

**Release date:** Sep 28, 2026

A small maintenance release. One user-visible bug fix, two message and
timing adjustments, and the supporting changes underneath them.

## Fixed

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

## Changed

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

## Under the Hood

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

---

## Credits

- jaudiotagger by jthink Ltd
- FontAwesome by Fonticons, Inc.

---

## License

Licensed under the MIT License

Copyright (c) 2026 Richard Korbla Adzido

Made with love in Ghana