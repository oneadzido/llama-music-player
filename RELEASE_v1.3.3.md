# Llama Music Player v1.3.3

**Release date:** Oct 13, 2026

## Overview

Three layout corrections: the theme dialog's top bar geometry is
aligned with the other dialogs, and two row-level margin issues in the
theme and language dialogs are resolved for both LTR and RTL locales.

## Fixed

- **Theme dialog's cancel button wraps its width correctly.** The
  invisible placeholder button in the theme dialog's top bar was
  configured with the Cancel string (`R.string.action_cancel`)
  instead of the Themes string (`R.string.action_themes`). Because
  the placeholder text matched the Cancel button rather than the
  Themes button, the top bar's horizontal layout miscalculated the
  space available for the Cancel button, preventing it from wrapping
  its content correctly. The placeholder now uses the Themes string,
  matching the Credits dialog and the Preset dialog exactly, so the
  Cancel button's width resolves naturally and the title remains
  centred.

- **Language dialog row alignment in Arabic locale.** In the language
  dialog, when a language is not the current one, the indicator text
  is cleared, but the indicator view retained its `marginStart`. In
  an RTL layout, this invisible margin pushed the language name away
  from the leading edge, creating an unwanted gap. The indicator is
  now set to `View.GONE` when it has no text to show, which removes
  the view and its margin from the layout calculation entirely. The
  language name now stretches to the edge, aligning correctly with
  the current language row.

- **Theme dialog's colour circle extra margin.** Similar to the
  language dialog issue, the theme dialog's indicator retained its
  `marginStart` when the row was not selected, even though its text
  was empty. This created an invisible gap between the colour
  preview circle and the trailing edge of the row. Setting the
  indicator to `View.GONE` when not selected removes this phantom
  margin. The gap between the theme name and the colour circle is
  preserved, and the circle now sits flush against the row's edge
  padding.

## Under the Hood

- `ThemeDialog.createPlaceholder()` now calls `setText` with
  `R.string.action_themes` instead of `R.string.action_cancel`.
- `LanguageDialog` and `ThemeDialog` now use `View.GONE` for their
  indicator views when the row is not selected, replacing the
  previous `setText("")` approach which left the view's margins in
  the layout tree.

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