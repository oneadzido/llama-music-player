# Llama Music Player v1.2.3

**Release date:** Oct 4, 2026

## Overview

Six fixes ship in this release, all in the playback layer.

The music-stream volume receiver is now registered only after the user
has opened the player. Previously the receiver was armed as soon as
the service was created, including on paths where the service starts
without the user opening the app — a Bluetooth connect, a sticky
restart, a media button. On one of those paths the service could be
left holding a prepared song in the `READY` state, and the next
hardware volume press started playback. The receiver is now registered
on bind and unregistered on destroy, so it exists only after the user
has opened the player.

A transient audio-focus loss is now remembered. When another app takes
focus for a bounded interval — a phone call, a navigation prompt, a
WhatsApp video status — the service pauses and remembers that it did
so. When focus is returned, playback resumes from where it paused. A
permanent loss is still respected: the service pauses and does not
resume on its own.

An interruption that offers to share the output — a navigation
prompt, a notification tone, a voice assistant reply — now ducks the
music instead of pausing it. The music keeps playing underneath at a
lower output level and returns to full volume when the interruption
ends. The system volume slider does not move.

The volume-up key now resumes playback from the foreground even when
the volume is already at the device maximum. At the maximum, the
framework emits no `VOLUME_CHANGED_ACTION`, so the service receiver
never fires; the volume key press is intercepted at the activity
level instead.

Two supporting changes ride alongside. The volume handler now reads
the current music-stream volume at the handling moment rather than
trusting the value carried by the broadcast. The storage observer no
longer starts a playlist scan when a non-audio file is written.

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

- **Playback resumes when a transient audio-focus loss ends** — When
  another app requested focus with `AUDIOFOCUS_GAIN_TRANSIENT` — a
  phone call, a navigation prompt, a WhatsApp video status — the
  service paused but did not remember why, so when focus returned it
  stayed paused. The service now sets `mPausedByTransientLoss` on a
  transient loss and, when `AUDIOFOCUS_GAIN` arrives while that flag
  is still set, resumes from the paused position. A permanent loss
  does not set the flag, so playback does not restart on its own
  when the user has moved to another app or another permanent audio
  source. The flag is cleared by any user action — resume, pause,
  seek, stop, or a volume drop to zero — so a stale flag cannot
  resume playback after the user has moved on.

- **Music ducks instead of pausing when an interruption offers to
  share the output** — Some interrupting apps request focus with
  `AUDIOFOCUS_GAIN_TRANSIENT_MAY_DUCK`. The framework delivers this
  to the losing app as
  `AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK`. A navigation prompt, a
  notification tone, or a voice assistant reply is what sends this.
  The service previously ignored this code, so the music kept
  playing at full volume while the interruption played over it. The
  service now lowers the MediaPlayer's own output volume to
  `DUCK_VOLUME_FRACTION = 0.2f` for the duration of the interruption
  and restores it to full when `AUDIOFOCUS_GAIN` arrives. Playback
  is not paused and the position is not lost. The system stream
  volume is not touched, so the volume slider does not move and the
  music-stream receiver does not fire.

- **Volume up at the device maximum resumes from the foreground** —
  When the music volume is already at the device maximum and the
  user presses volume up, the framework emits no
  `VOLUME_CHANGED_ACTION`: no value changed, so no broadcast fires.
  The service's receiver therefore never runs, and the resume path
  was unreachable in that case. `MusicPlayer` now overrides
  `dispatchKeyEvent` to observe `KEYCODE_VOLUME_UP` before the
  window system consumes it. When the volume is at the device
  maximum and the song is bound but paused, the activity resumes
  the song directly. The event is not consumed, so the system still
  processes the key as it normally does.

  The activity path covers the foreground case only. The framework
  does not deliver volume keys to a backgrounded activity, and no
  broadcast fires at the device maximum because no value changed.
  Every non-intrusive background channel has the same limitation:
  `VOLUME_CHANGED_ACTION` fires on a change that does not occur,
  `ContentObserver` on the volume setting has the same problem, and
  the media button action carries `KEYCODE_MEDIA_*`, not
  `KEYCODE_VOLUME_*`. The framework-supported paths that would reach
  a backgrounded app are all heavier than the case warrants — an
  accessibility service, a system overlay window, or a `MediaSession`
  remote volume provider that would move volume control away from the
  system stream. When the player is backgrounded and the song is
  deliberately paused, resuming goes through the notification's play
  action, which is delivered to the service through the media session
  in every screen state. That is the platform's intended affordance
  for the case, and it works from the lock screen and from the shade.

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

- `MusicService.handleAudioFocusChange(int)` distinguishes four
  cases. A permanent loss pauses and does not set any resume intent.
  A transient loss pauses and sets `mPausedByTransientLoss`. A duck
  request sets `mIsDucked` and applies the duck volume to the
  current player. `AUDIOFOCUS_GAIN` clears both flags: it restores
  the player's output volume if a duck was active, and resumes from
  the paused position if a transient pause was remembered and the
  state is still `PAUSED`.

- `MusicService.applyDuckVolume(MediaPlayer)` is the single point
  that reads `mIsDucked` and writes the resulting volume to a
  player. It runs on the player thread. It is called from
  `handleAudioFocusChange(int)` when a duck begins or ends, and from
  `beginTransition(int, int)` after each new MediaPlayer is created,
  so a track change that happens mid-duck does not momentarily
  restore the volume to full. The duck is a scale applied to the
  MediaPlayer's own output through `setVolume(float, float)`, not a
  change to the system stream volume, so the framework's
  `VOLUME_CHANGED_ACTION` does not fire and the music-stream
  receiver is not involved.

- `MusicService.isPausedOrReady()` returns true when the state
  machine holds a bound song that is not currently playing. It is
  the case a volume-key press starts playback from: the song is
  paused at a position, or prepared at its start and waiting for the
  play button. `MusicPlayer` reads it from the at-maximum volume-up
  path.

- `MusicPlayer.dispatchKeyEvent(KeyEvent)` observes
  `KEYCODE_VOLUME_UP` before the window system consumes it, and
  performs one side effect: when the volume is at the device maximum
  and the service reports `isPausedOrReady()`, the activity calls
  `MusicService.resume()`. The event is not consumed. Volume changes
  below the maximum are handled by the receiver in `MusicService`,
  which fires on the broadcast the system emits when the value
  changes. The activity path exists solely for the case where no
  value changes, and therefore no broadcast fires. It runs only
  while the activity is visible. The background resume path for a
  deliberately paused song is the notification's play action, which
  reaches `MediaSessionCompat.Callback.onPlay` and runs the same
  `doResume()` the activity path runs.

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