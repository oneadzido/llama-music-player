# Llama Music Player v1.2.8

**Release date:** Oct 10, 2026

## Overview

Six changes ship in this release — the permission-denied path now
leaves the app in a state the user can act on, the cold-start load
recovers from a MediaStore index that has not yet catalogued the
library, the play button never shows as live when the service is not
bound, the permission toast is shortened to fit the normal duration,
a headset disconnect pauses playback even when the player activity is
not alive, and the persisted last-playing flag has a single writer.

The permission flow has two outcomes and both need an exit. On a
grant, the app runs the initialization flow from the resume cycle
that follows the dialog. On a denial, the app applies the same empty
state it would show for a library with no songs, so the transport
controls are disabled and the user is not presented with buttons that
cannot do anything. A single resume-time gate owns "has initialization
run," so the flow cannot run twice for one permission decision and
cannot be stranded when the user grants the permission after an
earlier denial.

The cold-start load runs a MediaStore query that returns what the
index has catalogued. A file the index has not yet seen is not
returned by that query. When the query comes back empty and the user
has folders configured, the empty result is a gap in the index rather
than an empty library. The same folder-update scan the UPDATE button
runs is triggered in that case, and the cold-start overlay stays up
across both stages so the user sees one continuous load.

The play button's enabled state now tracks the service, not just the
library. Every tap on the button resolves to a service call; while the
service is not bound, the call is a no-op. A button that looks live
but does nothing is the wrong state to show.

The pause that a Bluetooth disconnect is meant to produce was owned by
the activity, not the service. The activity unregisters its receivers
in `onDestroy`, so the pause had no receiver once the activity had
been reclaimed by the OS or swiped from recents, and the music kept
playing through the phone's speaker with the headset gone. The service
owns the audio, so the service now owns the pause.

The persisted `last_playing` flag had two writers on the disconnect
path. A `GET_PLAYING_STATE` broadcast triggered a preference write from
both the activity and the service. The value is derived from the state
machine, and the state machine is the service's, so the service is the
writer.

## Fixed

- **Permissions denied now leaves the app in a state the user can
  act on** — The permission-denied branch hid the cold-start overlay
  and showed a toast, but never changed the UI state. The play button
  and the seek bar were enabled by the layout and stayed enabled,
  because the only method that disables them was never reached.

  The denial branch now applies the empty state. The title and artist
  slots carry their standard placeholders, the transport controls are
  disabled, the carousel shows the placeholder, both clock texts reset
  to `00:00`, and the cached last song is cleared. The screen is
  identical to the empty-library state because the two conditions are
  indistinguishable from the user's seat.

- **The cold-start load recovers from a MediaStore index gap** —
  When the cold-start query returned no songs but the user had folders
  configured, the app landed in the empty state and stopped. The
  user's folders were still selected, their songs were still in the
  database, and the empty state was a lie about the library's content.

  The load now runs the same folder-update scan the UPDATE button
  runs. The cold-start overlay stays up across both stages so the
  user sees one continuous load rather than a load followed by a
  correction.

- **The play button is no longer enabled when the service is not
  bound** — Every tap on the play button resolves to a service call.
  While the service is not bound, the call is a no-op. The button is
  now enabled only when the library holds songs **and** the service
  is bound.

- **The permission toast is shortened** — The denial message is now
  `Permissions required to read and play songs`.

- **A headset disconnect pauses playback even when the activity is
  not alive** — `BluetoothReceiver` broadcasts a pause request when
  the headset or A2DP profile disconnects. The request was consumed by
  `MusicPlayer`'s `mBluetoothPauseReceiver`, which is unregistered in
  `onDestroy()`. When the activity had been destroyed — the OS
  reclaiming it, the user swiping it from recents — the request had no
  receiver, and the music kept playing through the phone's speaker
  with the headset gone. The service was running and held the audio;
  it simply never registered for the pause.

  `MusicService` now owns the pause. It registers a receiver for
  `BLUETOOTH_PAUSE` in `onCreate()` and unregisters it in
  `onDestroy()`, so the pause fires for the service's whole lifetime
  rather than the activity's. The pause goes through the same
  `pause()` method the transport button uses, so the state machine,
  the persisted `last_playing` flag, the notification surface, and the
  `UPDATE_PLAYER` broadcast all come out the normal way.

- **The persisted last-playing flag has a single writer** —
  `BluetoothReceiver` sent a `GET_PLAYING_STATE` broadcast before the
  pause request. Two receivers in the process wrote
  `LlamaPrefs.last_playing` from it:
  `MusicPlayer.mGetPlayingStateReceiver` and
  `MusicService.mGetPlayingStateReceiver`, each calling
  `isPlaying()` on the same value. Two writers for one fact. The value
  is derived from the state machine, and the state machine is the
  service's, so the service is the writer. `MusicPlayer`'s
  `GET_PLAYING_STATE` receiver — whose only effect was the duplicate
  write — is removed. The service's own `GET_PLAYING_STATE` receiver
  remains, because `MetadataEditor` uses it for its own handshake and
  for the persistent write. `BluetoothReceiver` no longer sends
  `GET_PLAYING_STATE` on disconnect at all: the pause it gated is now
  handled by the service directly, and the metadata editor remains the
  broadcast's only sender for its own use.

## Under the Hood

- `MusicPlayer` gained `mInitializationAttempted` and
  `mPermissionsBlocked`. The first prevents the flow from running more
  than once per activity instance; the second latches on a denial and
  clears on a later grant, resetting the first so the flow runs again
  from a clean slate.

- `MusicPlayer.hasAllRequiredPermissions()` mirrors the request set
  without requesting anything. The resume-time retry uses it so a
  user who has already denied is not shown another dialog on every
  resume.

- `MusicPlayer.onResume` is the single owner of "has initialization
  run." The permission callback records the outcome; the resume reads
  it and drives the flow.

- `MusicPlayer.proceedWithInitialization()` skips the bind when the
  service is already bound — the recovery path after a permission
  restore — and calls `loadFullPlaylistInBackground` directly.

- `MusicPlayer.loadFullPlaylistInBackground()` triggers
  `triggerAutomaticFolderUpdate()` on an empty result with folders
  configured. The auto-update's completion or error callback hides
  the overlay.

- `MusicPlayer.updatePlayPauseButton()` checks both the library and
  the service.

- `MusicService` gained `ACTION_BLUETOOTH_PAUSE` and the field
  `mBluetoothPauseReceiver`. `registerBluetoothPauseReceiver()`
  registers it in `onCreate()` after `registerLibraryResetReceiver()`;
  `unregisterReceiverSafely(mBluetoothPauseReceiver)` clears it in
  `onDestroy()`. The receiver's `onReceive` calls `pause()` when the
  action matches.

- `MusicPlayer` dropped the field `mBluetoothPauseReceiver` and the
  field `mGetPlayingStateReceiver`; dropped their
  `registerReceiverSafely` calls in `registerReceivers()`; dropped
  their `unregisterReceiverSafely` calls in `onDestroy()`; and
  dropped the `KEY_LAST_PLAYING` constant, which the get-state
  receiver had been the only writer of.

- `BluetoothReceiver.handleBluetoothDisconnect()` no longer sends
  `GET_PLAYING_STATE`. The `ACTION_GET_PLAYING_STATE` constant is
  removed.

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