# Llama Music Player v1.2.9

**Release date:** Oct 10, 2026

### Overview

This release brings permission-aware state handling to folder management
and playback. The required runtime permissions are defined in one place,
and controls now reflect whether the app can perform the action they
represent. Folder changes depend on permission and update state; playback
controls depend on permission, service readiness, and the contents of the
music library.

### Fixed

- **Folder actions reflect permission and update state** — UPDATE and
  per-folder REMOVE buttons are disabled when the required permissions are
  missing or an update is already running. IMPORT FOLDER button remains available
  when no update is in progress, since the system folder picker provides a separate
  way to grant folder access. When permissions are restored and the user
  returns to the app, the folder controls are refreshed to reflect the
  current state.

- **Playback controls reflect whether playback is available** — Play,
  previous, next, repeat, shuffle, and seek are disabled when the app lacks
  required permissions, the playback service is not bound, or the library
  has no songs. This also prevents theme application or a stale in-memory
  song count from re-enabling controls after permission revocation.

### Under the Hood

- `PermissionChecker` is the single definition of the required
  runtime-permission set. `getMissing(Context)` supplies permissions
  for the request flow, and `allGranted(Context)` checks the same
  version-gated set. `MusicPlayer.checkPermissionsAndInitialize()`,
  `MusicPlayer.onResume()`, and `FolderManager` use this shared source
  instead of maintaining separate permission lists or checks.
  `MusicPlayer.hasAllRequiredPermissions()` was removed because it
  only duplicated the shared check. The permission flag was renamed
  from `mPermissionsBlocked` to `mPermissionsDenied`; when permissions
  are granted again, the initialization-attempt state is reset as
  needed so initialization can run from a clean state.

- `FolderManager.areFolderChangesAllowed()` combines
  `PermissionChecker.allGranted(this)` with `!isUpdating()`. The UPDATE
  button and row-level REMOVE buttons use this same decision.
  `updateFolderButtons()` refreshes the adapter against that answer,
  while IMPORT FOLDER button continues to depend only on whether an
  update is running.
  `FolderAdapter` receives a `RemoveButtonStateSource` from
  the activity and queries it when configuring each REMOVE button,
  rather than reading the navigation controller's disabled-action
  state or caching an earlier result.

- `MusicPlayer.canEnableTransportControls()` combines the permission
  check, service availability and binding state, and library contents.
  `MusicPlayer.refreshTransportControls()` is the single writer of
  every transport control's enabled state, label, and active tint. When
  controls are available, it reads the play state, repeat mode, and
  shuffle state from `MusicService`. It also updates the seek bar's
  enabled state and tint without resetting its progress or range, so a
  refresh does not interrupt a seek gesture.

- The previous separate writers —
  `updatePlayPauseButton()`, `updateRepeatButton()`,
  `updateShuffleButton()`, `updateControlButtonsState()`, and
  `enableAllControls()` — were removed and their responsibilities
  moved into `refreshTransportControls()`. Initialization, service
  connection, resume, song/UI updates, empty-state handling, theme
  application, playlist loading, player and library broadcasts, and
  repeat/shuffle actions now refresh controls through that method.
  `applyNoSongState()` retains responsibility for resetting the seek
  bar's progress and range, then delegates control state to the shared
  refresh. The field and class documentation now identify this single
  ownership point and its permission check.

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