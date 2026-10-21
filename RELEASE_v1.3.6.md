# Llama Music Player v1.3.6

**Release date:** Oct 21, 2026

### Overview

Two fixes ship in this release: a layout consistency pass for the
Metadata Editor, and a notification state flush for audio focus
changes.

The vertical spacing in the Metadata Editor previously relied on a mix
of parent container padding and child view padding. The spacing is now
driven entirely by explicit `layout_marginTop` values on the section
labels, guaranteeing uniform bounding boxes and consistent behavior
across all screen sizes.

The notification surface previously derived its play/pause icon from
the service's state machine, but the service did not flush the
notification controller on every audio focus change. A focus event
that did not change the state — a duck, a gain without a remembered
pause, a loss while already paused — could leave the notification
showing the state from the previous transition. The service now pokes
the controller's awareness poll on every focus change, ensuring the
icon always reflects the current state.

### Fixed

- **Metadata Editor Label Bounding Boxes** — The "Title" label
  relied on its parent's padding, while "Artist" and "Album" used
  their own `paddingTop`, and the grouped rows ("Year/Genre",
  "Track/Info", "Lyrics") used parent `paddingTop`. This resulted in
  inconsistent layout bounds for the `TextView` elements. All
  section labels and rows now use `layout_marginTop="12dp"` to
  establish the gap, removing the reliance on parent paddings.
- **Metadata Editor Lyrics Label Bottom Padding** — The bottom padding
  of the "Lyrics" label was 8dp, while every other label used 4dp. It
  has been reduced to 4dp to match the rest of the form.
- **Notification Play/Pause State During Audio Focus Changes** — The
  notification controller is now flushed on every audio focus change.
  Previously, ducking (`AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK`) or a
  transient loss that arrived during `PREPARING` could leave the
  notification displaying a stale play/pause icon. The service now
  pokes the controller on every branch of `handleAudioFocusChange`,
  and correctly prevents auto-starting a prepared track if focus was
  lost during the prepare.
- **Notification State During Prepare** — When a permanent or
  transient audio focus loss arrives while the state machine is in
  `PREPARING`, the service now sets `mPauseAfterPrepare` and clears
  `mAutoStartOnPrepared`. This prevents the track from starting
  playback on a lost focus.

### Changed

- **Metadata Editor Album Art Section** — Removed the
  `paddingBottom="8dp"` from the album art container. The space
  between the album art and the "Title" label is now provided by the
  Title label's `layout_marginTop`.
- **Metadata Editor Editable Fields Container** — Removed the
  `paddingTop="4dp"` from the parent container holding the editable
  fields. The top spacing is now entirely managed by the Title label.

### Under the Hood

- `activity_metadata_editor.xml` — Replaced `paddingTop` with
  `layout_marginTop="12dp"` for the "Title", "Artist", "Album" text
  views, and the "Year/Genre", "Track/Info", and "Lyrics" row
  containers. Removed the `paddingTop` from the "Editable Fields"
  `LinearLayout`, removed the `paddingBottom` from the "Album Art
  Section" `LinearLayout`, and reduced the "Lyrics" label's
  `paddingBottom` to `4dp`.
- `MusicService.java` — Updated `handleAudioFocusChange(int)` to call
  `mNotificationController.pokeAwareness()` on every branch. Added
  handling for the `PREPARING` state during `AUDIOFOCUS_LOSS` and
  `AUDIOFOCUS_LOSS_TRANSIENT` to set `mPauseAfterPrepare` and clear
  `mAutoStartOnPrepared`.

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
| Language | Tap LLAMA button on main screen, then the language button in the version header |
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