# Llama Music Player v1.1.8

**Release date:** Sep 27, 2026

## What's New

This release restructures roughly half the non-UI classes. Four subsystems were rewritten end to end, two classes were renamed, and the framework media session classes were migrated to the AndroidX support library. The user-facing behavior is largely unchanged; the code underneath is not.

- **Notification ownership inverted** — `NotificationController` replaces `NotificationService`. The controller polls the service through a state-source interface instead of receiving pushed state, and is now the sole writer of the media session metadata.
- **Media session and notification committed together** — one `RenderSnapshot`, one main-thread turn, two projections. The carousel and the notification can no longer show different songs on a rapid switch.
- **Operation stack replaces the single-owner overlay token** — `ToastManager` renders from an ordered stack of operations. An operation below the top is retained but not shown; a cancelled operation is removed as a terminal state, and the overlay cannot freeze.
- **Folder-update cycle overlay ownership made strictly sequential** — folder sync, playlist rebuild, and genre loading each own the persistent overlay for the duration of their own work. One operation per phase, pushed and popped by its owner, no operation ever left on the stack underneath another.
- **Empty library leaves no trace** — the media session is created inactive and the service does not promote to foreground when the playlist is empty. A first launch with no songs shows nothing in the status bar or the media carousel.
- **Library reset added** — a first-class `PlaylistManager.resetLibrary()` operation with its own write barrier, invalidation broadcast, and completion signal.
- **Media session migrated to AndroidX** — `MediaSessionCompat`, `PlaybackStateCompat`, `MediaMetadataCompat`, and `MediaButtonReceiver` replace the framework classes.
- **Bluetooth resume rebuilt** — a dedicated `ACTION_BLUETOOTH_RESUME` service action replaces the generic play broadcast. A2DP disconnect now pauses playback.
- **Result types replace ad-hoc returns** — `MetadataReader`, `MetadataWriter`, and `CharacterMapper` now return typed results with exhaustive status enums.
- **Equalizer settings made authoritative in `SharedPreferences`** — getters read what setters wrote, regardless of any rounding the underlying effect applies.

---

## Architecture Overhaul

### Renamed and Restructured

| v1.1.7 | v1.1.8 | What changed |
|---|---|---|
| `NotificationService` | `NotificationController` | Inverted ownership. v1.1.7 pushed state in and built notifications reactively. v1.1.8 polls a `ServiceStateSource`, owns two channels, and commits changes through an awareness loop. It is also the sole writer of the media session metadata. Not a rename — a different architecture. |
| `PlaylistService` | `PlaylistManager` | Rename plus ownership change. It now owns the full library reset, owns the genre-loading overlay for its entire lifetime, and no longer shares scan lifecycle with `StorageObserver`. |

### Framework → AndroidX Support Library

Every media session type moved from the framework namespace to the compat namespace:

| v1.1.7 | v1.1.8 |
|---|---|
| `android.media.session.MediaSession` | `android.support.v4.media.session.MediaSessionCompat` |
| `android.media.session.PlaybackState` | `android.support.v4.media.session.PlaybackStateCompat` |
| `android.media.MediaMetadata` | `android.support.v4.media.MediaMetadataCompat` |
| Custom `MEDIA_BUTTON` handling in `BluetoothReceiver` | `androidx.media.session.MediaButtonReceiver` plus session dispatch |

This is driven by the `androidx.media:media` 1.0.0 → 1.8.0 bump. The framework session classes work in isolation, but they have no media button receiver that resolves by intent filter and they do not interoperate with the current notification `MediaStyle` the way the compat classes do. `MusicService` now registers a `MediaButtonReceiver` component name on its session and handles `Intent.ACTION_MEDIA_BUTTON` by dispatching the `KeyEvent` through the active session.

### Result Types Replace Ad-Hoc Returns

Three classes that previously returned nullable fields with implicit "not present" semantics now return typed results with exhaustive status enums.

| v1.1.7 | v1.1.8 |
|---|---|
| `MetadataReader.MetadataBundle` with null/empty checks | `MetadataBundle.status` (`SUCCESS` / `FILE_UNREADABLE` / `PARSE_FAILED`) plus `hasAlbumArt()` |
| `MetadataWriter.WriteResult` with `boolean success` + nullable message | `WriteResult.status` enum, `isSuccess()`, `isFileUnchanged()` |
| `CharacterMapper.SongInfo` with nullable fields | `CharacterMapper.ParseResult` with `Status`, `hasRealGenre()`, `hasRealArtist()`, `hasRealAlbum()` |

Every caller of these three classes was rewritten to branch on the status. Error messages are now used only for logging and display. Control flow never inspects a diagnostic string.

`CharacterMapper` also stopped parsing each file once per field. `getTitle`, `getArtist`, `getAlbum`, and `getGenre` each call `parseSongInfo` once and read the result.

### Data Layer Restructured

- `SongDatabase` gained a `Table` enum. Every read, insert, delete, and clear is now a single parameterized implementation. The two tables shared a schema before; now they share code.
- `SongDatabase.RowReader` caches column indexes per query. `Cursor.getColumnIndex()` walks the cursor's column-name array on every call; the previous code resolved six columns per row over a multi-thousand-row result, which was the dominant cost of a full-table read.
- `MusicLibrary`'s constructor is now asynchronous. The initial database read was previously synchronous on the calling thread.
- `MusicLibrary.refreshCache()` is now the sole writer of the in-memory song list and reads the database under the write lock.
- `MusicLibrary.invalidate()` is new. It is the reset-path counterpart to `refreshCache()`: it clears the in-memory lists, drops the loaded flag, and re-reads the folder set from `SharedPreferences`. Two different cache-invalidation verbs for two different situations.

### New Operation: Library Reset

`PlaylistManager.resetLibrary(ResetMode, ResetCallback)` is a first-class operation that did not exist in any form in v1.1.7.

- **Admission.** Refuses when a scan or another reset is in flight.
- **Write barrier.** On the executor, clears the songs table, the loose-tracks table, the SAF folder mappings, and the folder-selection preference. Everything above is content.
- **Invalidation broadcast.** After the barrier commits, a single `LIBRARY_RESET` broadcast tells every component to invalidate its song-derived state. The reset does not know who the owners are.
- **Completion signal.** The callback fires after the barrier and after the broadcast dispatches. Work ends when its work completes, not when a timer expires.

Two modes: `ResetMode.LIBRARY` preserves user settings, `ResetMode.FACTORY` clears them. The equalizer settings, theme, repeat mode, shuffle flag, and sort mode are preserved in the LIBRARY case.

Four receivers register for `LIBRARY_RESET`: `MusicPlayer`, `PlaylistActivity`, `FolderManager`, and `MusicService`. Each invalidates its own state. `MusicService`'s receiver tears itself down through `transitionToIdle()`, clears the media session metadata, cancels the playback notification immediately through `NotificationController.clearPlaybackNotification()`, and clears the resolved notification art, the art cache, the static audio-session identifier, and the transport-action debounce map. `PlaylistActivity`'s receiver additionally clears its own genre-loading flag and pending-sort flag, because the reset cancels any in-flight genre load through its own operation cancellation and the genre load's completion broadcast therefore never arrives.

A short-lived background thread sweeps stale `temp_audio_*` and `backup_*` files from the cache directory after the barrier commits.

### Notification Ownership Inverted

In v1.1.7, `NotificationService` was attached to `MusicService` and built notifications from state pushed to it. In v1.1.8, the direction reverses.

`MusicService` exposes a `NotificationController.ServiceStateSource` that returns a `RenderSnapshot`. `NotificationController` polls it every 100 ms and commits the difference directly. The notification is a live projection of the service, delayed by at most one poll interval.

`RenderSnapshot` carries title, artist, album, path, play flag, duration, art, and an `artResolved` flag — no position, no timestamp. Position updates flow through a separate broadcast to the activity. The snapshot therefore changes only on discrete events: a committed song transition, a committed play/pause flip, or an art decode resolving. A rapid-skip burst produces exactly one snapshot at the end.

Two holds preserve the source's truth during a transition:

- **PREPARING.** The source returns null while the state machine is mid-transition. The renderer holds the last committed state. This is not lag; it is the source reporting that nothing new has committed yet.
- **Art decode.** When a new song arrives and its art is still being decoded, the entire previous render is held — title, artist, and art together — rather than displaying the incoming song's title over the outgoing song's cover. Mixed data reads as a bug; coherent staleness reads as lag.

The controller owns two notification identifiers on two channels: playback (`music_playback_channel`) and library (`music_library_channel`). Only the playback notification is eligible for foreground promotion. The library notification mirrors `ToastManager`'s persistent overlay and splits its message into a primary and secondary line on the first `"..."` separator, with an elapsed-time chronometer driven entirely by the system UI.

### Single Writer for the Media Session

In v1.1.7, `MusicService` wrote the media session's metadata eagerly on `onPrepared` and `onPlaybackStarted`, while the notification held its render until the incoming song's art resolved. On a rapid switch `A → B → C`, the carousel showed `C` while the notification still showed `A`, and because the notification's `MediaStyle` is bound to the same session, its transport buttons operated on `C`.

v1.1.8 removes every direct session write from `MusicService`. `MediaMetadataCompat.setMetadata` and `PlaybackStateCompat.setPlaybackState` are now called only from `NotificationController.applySnapshot`, in the same main-thread turn that commits the notification. Both surfaces are projections of one `RenderSnapshot` committed once, so they cannot disagree.

The playback state write is gated on the same art-resolution condition the metadata write uses. During the hold window, the session keeps the previous song's position; when art resolves, `applyToMediaSession` writes the metadata and then calls `updateMediaSessionPlaybackState` so position and identity flip together.

`MusicService`'s art prefetch pokes the controller's awareness poll at the moment the decode resolves, so the commit that releases a held render happens on the same main-thread turn as the resolve rather than on the next scheduled tick.

### The Operation Stack

v1.1.7 held one persistent overlay and one operation ID. That model encodes "one overlay, one owner" and cannot represent two concurrent operations. v1.1.8 replaces it with an ordered stack of operation IDs, each carrying its own message and heartbeat timestamp.

- An operation is pushed by `showPersistentOverlay(activity, message, operationId)`. Pushing an operation that is already on the stack moves it to the top and updates its message.
- The operation at the top is the one rendered. Operations below the top have their messages retained and surface automatically when the operations above them end.
- `hidePersistentOverlay(operationId)` and `cancelOperation(operationId)` are the same operation: removing an entry from the stack. A cancelled operation is a terminal state — a long-running loop observes `isOperationRunning(operationId)` returning false and exits without reaching its completion path.
- The view map and the operation stack are separate stores with separate lifetimes. Removing an overlay view does not remove its operation; removing an operation does not by itself remove any view. This is what lets an operation whose activity was destroyed continue running until a new activity resumes and rebuilds the overlay from the top of the stack.
- Two paths clear the entire stack: the unconditional force-hide and the stuck-overlay recovery pass. Both own every operation by definition. No other path touches the stack.

`recomputeStateFromMap()` now considers the stack when deciding the state: an empty map with a non-empty stack stays in `SHOWING_PERSISTENT`, because the operation wants an overlay and the overlay is waiting for a surface to attach to.

### The Folder-Update Cycle

The folder-update cycle is a sequence of three overlays, each owned by the component doing the work it covers, pushed and popped by that component alone. At no point does the operation stack hold more than one operation, and no operation is ever left on the stack underneath another.

**Folder sync.** Owned by `FolderManager`. `FolderManager.performPlaylistUpdate` pushes `FOLDER_SYNC` on the scan's first progress callback, updates its message on each subsequent callback, and pops it on scan completion on every exit path — success, error, and full-wipe completion. When `FOLDER_SYNC` is popped, `FolderManager`'s involvement in the overlay stack for that cycle ends.

**Playlist rebuild.** Owned by `MusicPlayer`. The `UPDATE_PLAYLIST_FROM_FOLDERS` receiver pushes `PLAYLIST_REBUILD` into whichever activity is currently resumed, using `ToastManager.getCurrentActivity()`, before starting the library rebuild on its metadata loader. The overlay is heartbeated by a self-scheduling runnable for the duration of the rebuild. When the rebuild completes, the same main-thread continuation that hands the new playlist to the service pops the overlay and broadcasts `PlaylistManager.ACTION_TRIGGER_GENRE_LOADING`.

Pushing the overlay with the currently-resumed activity rather than with `MusicPlayer` itself is what makes it visible to the user mid-cycle. `ToastManager.onActivityResumed` already reattaches an active overlay whenever an activity resumes, so the rebuild overlay follows the user across navigation the same way the cold-start `LIBRARY_LOAD` overlay does.

**Genre loading.** Owned by `PlaylistManager`. `triggerGenreLoading` pushes `GENRE_UPDATE`, heartbeats it through a token-cancellable ticker that is independent of the ID3 read loop, and pops it when the loop completes. The loop itself runs on `PlaylistManager`'s executor; the overlay's own heartbeat runs on a dedicated main-thread handler so a slow ID3 read cannot starve the heartbeat and trip the stuck detector.

**Why the sequence is safe.** Because each operation is popped by its owner before the next one is pushed, `popOperationInternal` never finds a lower operation to promote back to the top. When the last operation in the cycle pops, `newTopId` is null, the overlay is torn down through the empty-stack path, and the status-bar mirror is dismissed. This is what prevents the folder-sync overlay from lingering after genre loading finishes.

### `ToastManager` Overlay Model Replaced

v1.1.7 used a single overlay key per activity. v1.1.8 uses two namespaces: persistent overlays keyed by the activity key, temporary toasts keyed by a prefixed variant of the same key. The namespaces never overlap, so a temporary toast and a persistent overlay for the same activity coexist.

A temporary toast now draws on top of a persistent overlay rather than replacing it. When a persistent overlay is active, the state machine stays in `SHOWING_PERSISTENT`; the temporary toast dismisses on its own timer without touching the persistent message or operation ID.

A per-activity temporary toast replaces the previous temporary toast for that activity and cancels its pending auto-dismiss. A temporary toast in a different activity is independent.

`NotificationController` mirrors every persistent overlay to the status bar. The operation ID is passed through as an ownership token, so a hide or update that arrives after another operation has taken over the overlay is refused.

### Empty Library Leaves Nothing Behind

Two changes make a first launch with no songs invisible to the status bar and the media carousel.

- The media session is created **inactive** by `setupMediaSession`. Activation is owned by the playback state machine: it happens at the READY transition of `onPrepared`, at the PLAYING transitions of `onPlaybackStarted` and `onPlaybackResumed`, and in the READY branch of `togglePlayPause`. `transitionToIdle` deactivates it. The session is active if and only if a track is bound.
- Foreground promotion in `onStartCommand` is gated on the playlist being non-empty. A cold start with no content does not post a resting notification and does not pull the service into a foreground state it has no reason to hold. As soon as `setPlaylist` loads a track and the state machine reaches READY, the awareness poll sees a non-null snapshot and posts the notification; foreground promotion happens when playback actually starts.

---

## Behavioral Changes

### Bluetooth Resume

`BluetoothReceiver` no longer broadcasts a generic play request on reconnect. When a profile connects and there is prior playback state with a valid last song path, it starts `MusicService` with a dedicated `ACTION_BLUETOOTH_RESUME` action. The service prepares the last song, resumes playback, and reloads the full library in the background.

The receiver also now listens for both headset and A2DP connection-state changes. A headset that loses only A2DP while remaining paired has moved the user's audio elsewhere; the disconnect handler pauses to reflect that.

Media button handling is removed from this receiver entirely. Media buttons flow through `MediaButtonReceiver` and the active `MediaSessionCompat`.

The receiver is gated in the manifest by `android:permission="android.permission.BLUETOOTH"`, so only processes that hold BLUETOOTH themselves can deliver a matching broadcast.

### Metadata Editor Broadcast

The editor now publishes a single `METADATA_UPDATED` broadcast after a successful write. The previous two-broadcast sequence (`UPDATE_SONG_IN_PLAYLIST` followed by `FORCE_UPDATE_PLAYER`) is gone. Each consumer reacts to the one broadcast according to its own concerns. The service's receiver for this broadcast pokes the controller's awareness poll, so a metadata edit reaches both the notification and the session in one turn.

### Equalizer Settings Persistence

`SharedPreferences` is now the authoritative store for every user-facing equalizer setting.

- `getPitch()`, `getSpeed()`, `getBassBoost()`, `getSurround()`, and `isEnabled()` read the persisted value, not the audio-effect object's rounded value.
- `setEnabled()`, `setPitch()`, `setSpeed()`, `setBassBoost()`, and `setSurround()` write to `SharedPreferences` before touching the effect object.

A caller that sets a value and reads it back receives the value it set, regardless of any rounding the underlying effect applies. `EqualizerManager.release()` saves the settings to `SharedPreferences` before tearing the effects down; it never removes those keys.

### Playback State Machine

- New `mStartToken` counter identifies the current start or resume continuation. A start or resume whose token has been superseded is inert, matching the existing generation gate for prepare and completion callbacks.
- New `mPauseAfterPrepare` flag records a pause that arrived during PREPARING and honours it at commit.
- `getCurrentSong()` now reports `mCurrentIndex`, the index the player is actually bound to. The two-index model is unchanged.
- `playNextSong()` and `playPreviousSong()` base their computation on `mCurrentIndex`.
- The media session is active only while a track is bound (PLAYING, PAUSED, or READY) and inactive in IDLE.
- The idle playback state maps to `STATE_NONE` rather than `STATE_STOPPED`, so a stopped-but-present entry no longer lingers in the system media carousel.

### Folder Sync and Genre Loading

- `FolderManager` uses a `PendingChanges` holder for staged removals. A batch of removals produces a single playlist rebuild.
- Genre loading runs entirely on `PlaylistManager`'s executor. The prologue — reading the full song list and filtering out rows that already have a genre — now runs on the executor rather than on the calling thread. On a multi-thousand-song library, the prologue previously blocked the main thread before the genre overlay could render.
- Genre loading's heartbeat runs on a dedicated main-thread ticker with token-based cancellation, so a slow ID3 read cannot starve the heartbeat and trip the stuck detector.
- Genre loading is cancellable as an operation. A reset cancels it by removing its entry from the stack; the loop observes the removal and exits without emitting `GENRE_UPDATE/COMPLETE`. `PlaylistActivity`'s `LIBRARY_RESET` receiver clears its own genre-loading flag in the same turn, so the activity does not suppress refreshes forever when the completion broadcast never arrives.
- Folder sync, playlist rebuild, and genre loading each own a distinct persistent overlay for the duration of their own work. Each is pushed and popped by a single owner. No operation is ever left on the stack underneath another, so no operation is ever promoted back to the top when a later operation pops.

### `NavigationController`

`Operation` now declares its own liveness contract through `getHeartbeatIntervalMs()`, `getMissedHeartbeatTolerance()`, and `getLivenessWindowMs()`. The health check enforces each operation's declared contract. An operation runs for as long as it honours its own declaration.

---

## Build and Platform

- Java 11 upgraded to **Java 17** for source and target compatibility.
- `encoding 'UTF-8'` set explicitly so source files decode identically across machines and CI runners.
- `androidx.core:core` upgraded from 1.0.0 to **1.13.1** for the compatibility surface required by `androidx.media` 1.8.0 under compileSdk 35.
- `androidx.media:media` upgraded from 1.0.0 to **1.8.0** for `MediaSessionCompat`, `MediaButtonReceiver`, and the compat notification media style.
- `viewBinding` explicitly disabled in the build features block. Every activity uses `findViewById`.

### Manifest

- **MediaButtonReceiver** registered as an exported receiver for `android.intent.action.MEDIA_BUTTON`.
- **MusicService** intent filter reduced to `android.intent.action.MEDIA_BUTTON`. The browser-service action and the duplicate media-button action are gone.
- **BluetoothReceiver** intent filters updated to the profile-specific actions and gated by `android:permission="android.permission.BLUETOOTH"`.
- `android:usesCleartextTraffic` removed.
- `android.hardware.microphone` uses-feature removed.
- `parentActivityName` added to every non-launcher activity.

---

## Under the Hood

### Storage Observer

`StorageObserver` no longer owns the `NavigationController` lifecycle for its scans. It asks `PlaylistManager` to run a scan; the manager owns the begin and end. The observer reads the controller's group state to learn whether a scan is running. The observer's view of the update state is the controller's view by construction.

### Metadata Cache File Extensions

`MetadataReader` and `MetadataWriter` now determine the cache file's extension from the source document's display name. Naming every cached file `.mp3` would force jaudiotagger's MP3 reader onto FLAC, OGG, M4A, and MP4 sources.

### `PlaylistAdapter`

- `setCurrentPlayingPath()` is now idempotent. A repeated assignment of the same path is a no-op, so a redundant call does not rebind the highlighted row twice.
- `shouldBulkReplace()` bypasses DiffUtil's Myers pass when more than half the rows are added, removed, or reordered. The animation DiffUtil buys would cost more than it is worth on a cold start, a first import, or a full rebuild.
- `setScrolling()` defers genre updates while the user is scrolling. Applying a payload-based rebind mid-scroll would call `notifyItemChanged()` on a visible row while the scroll animation is still settling, and the current row would appear to jump out of its own position.

### Theme

All three dialogs (`PresetDialog`, `ThemeDialog`, `CreditsDialog`) now use the same top-bar geometry. The CANCEL button measures and positions identically across all three.

### Receiver Registration

Every receiver registered by an activity or by `ToastManager` uses `ContextCompat.registerReceiver` with `RECEIVER_NOT_EXPORTED`. These receivers listen only for broadcasts sent by the app itself.

### `SYNC_STARTED` Relay

The activity's `SYNC_STARTED` receiver is a relay, not an owner. When an overlay is already on screen, it applies the incoming message under the operation ID that currently owns the overlay; when no overlay is on screen, it shows one only when the sender named an owner. The activity never invents an operation ID, because a fabricated ID creates an overlay that no operation can later hide.

---

## System Requirements

- **Minimum SDK:** Android 6.0 (API 23)
- **Target SDK:** Android 15 (API 35)

---

## Download

Download the APK from the Assets section of this release.

---

## Installation

1. Download the APK file
2. Enable "Unknown Sources" in your device settings
3. Open the APK file and tap "Install"

---

## Credits

- jaudiotagger by jthink Ltd
- FontAwesome by Fonticons, Inc.

---

## License

Licensed under the MIT License

Copyright (c) 2026 Richard Korbla Adzido

Made with love in Ghana