# Llama Music Player v1.2.3

**Release date:** Oct 4, 2026

## Overview

The music-stream volume receiver is now registered only after the user
has opened the player. Previously the receiver was armed as soon as
the service was created, including on paths where the service starts
without the user opening the app — a Bluetooth connect, a sticky
restart, a media button. On one of those paths the service could be
left holding a prepared song in the `READY` state, and the next
hardware volume press started playback. The receiver is now registered
on bind and unregistered on destroy, so it exists only after the user
has opened the player.

Two supporting changes ship alongside the fix. The volume handler now
reads the current music-stream volume at the handling moment rather
than trusting the value carried by the broadcast. The storage observer
no longer starts a playlist scan when a non-audio file is written.

## Fixed

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

## Under the Hood

- `MusicService.registerMusicVolumeReceiver()` is called from
  `onBind(Intent)` and nowhere else. It is idempotent, so a repeat
  bind is a no-op, and a registration failure leaves
  `mMusicVolumeReceiver` null so the next bind retries.
  `MusicService.unregisterMusicVolumeReceiver()` in `onDestroy()`
  remains the single teardown path. There is no separate session
  flag: the receiver's own field is the session state, so there is
  nothing for a second representation to drift against.

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