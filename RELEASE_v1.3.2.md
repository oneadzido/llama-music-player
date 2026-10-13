# Llama Music Player v1.3.2

**Release date:** Oct 13, 2026

## Overview

One resource correction for localization: the equalizer's zero-value
strings for semitones and decibels are now parameterized so they
format consistently with their positive and negative counterparts.

## Fixed

- **Equalizer zero-value strings are parameterized.**
  `equalizer_semitones_zero` is now `%1$.1f st` and
  `equalizer_db_zero` is now `%1$.1f dB`. Previously these strings
  hardcoded `0.0 st` and `0.0 dB`. The format specifier replaces the
  hardcoded value, so the zero value is formatted by the same code
  path that formats the positive and negative variants, and the
  decimal separator follows the app's language through the JVM
  default locale. A translator no longer maintains a separate literal
  for zero.

## Under the Hood

- `equalizer_semitones_zero` and `equalizer_db_zero` are updated to
  use `%1$.1f` with their respective units in the locale files that
  carried the hardcoded literals. The positive and negative variants
  already used the specifier; the zero variants now match.

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