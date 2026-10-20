# Llama Music Player v1.3.5

**Release date:** Oct 20, 2026

### Overview

A visual alignment pass across every activity and dialog. String-based
separators have been replaced with explicit 8dp start margins in headers
and list rows, guaranteeing a visible, consistent gap across all screen
sizes and languages. The Music Player timers, the Credits dialog version
header, and the Equalizer frequency bands have been adjusted to anchor
correctly to the 16dp edge and maintain consistent spacing in
right-to-left languages like Arabic. A layout wrapping issue in the
metadata editor's header has also been resolved.

### Fixed

- **Music Player Timer Alignment** — The current time and total
  duration text views now anchor flush to the left and right ends of
  the seek bar in every language. A flexible spacer between them
  ensures they remain separated without overflowing their bounding
  boxes.

- **Credits Dialog Version Header** — The `VERSION` label, the
  language button, and the version number now use relative start and
  end gravity. This anchors the version number to the 16dp trailing
  edge in all languages, preventing it from pushing toward the
  language button in right-to-left layouts.

- **Equalizer Frequency Bands Spacing** — The dynamically generated
  frequency bands now apply the same 8dp margin between the frequency
  label and the value text (e.g., "60 Hz" and "+2.0 dB") that the
  static effect controls carry.

- **Metadata Editor Header Wrapping** — The file format label
  (e.g., "MP3 file") in the metadata editor's header now sizes itself
  to its content and is anchored to the trailing edge with a consistent
  8dp start margin.

### Changed

**Header and List Separators.** The codebase has moved away from
string-based separators (`" "`) in Java and now uses explicit visual
margins directly in the XML layouts and programmatic UI builders. This
ensures the spacing is consistent and not subject to text-rendering
quirks.

- **Playlist Header** — The song count and the `SORT` button now sit
  after the sort mode label, separated by an 8dp start margin.
- **Folder Manager Header** — The folder count now sits after the list
  header, separated by an 8dp start margin.
- **Folder List Rows** — The folder icon, the folder name, and the
  `✕` button are now separated by 8dp start margins in
  `item_folder.xml`.
- **Metadata Editor Header** — The file format now sits after the
  mode label, separated by an 8dp start margin.
- **Credits Version Header** — The `VERSION` label, the language
  button, and the version number are now visually separated by their
  respective layout weights and gravity.
- **Language Dialog Header** — The language count now sits after
  the list header, separated by an 8dp start margin.
- **Preset Dialog Header** — The preset count now sits after the
  list header, separated by an 8dp start margin.
- **Theme Dialog Header** — The theme count now sits after the
  list header, separated by an 8dp start margin.

### Under the Hood

- `activity_playlist.xml`, `activity_folder_manager.xml`,
  `item_folder.xml`, `activity_metadata_editor.xml` — Added
  `android:layout_marginStart="8dp"` to adjacent elements in the
  headers and list rows.
- `activity_music_player.xml` — Replaced the weighted timer text
  views with `wrap_content` text views and a flexible `<Space>`
  element to separate them. Added `start` and `end` gravity to the
  timers.
- `EqualizerActivity.java` — Added `setMarginStart(dpToPx(8))` to the
  value text in `createSingleBandControl` to match the XML effect
  controls.
- `CreditsDialog.java` — Changed the version header text views to use
  relative `START`/`END` gravity and removed manual RTL view swapping
  from `createVersionHeader()`.
- `PlaylistActivity.java`, `FolderManager.java`, `FolderAdapter.java`,
  `MetadataEditor.java`, `LanguageDialog.java`, `PresetDialog.java`,
  `ThemeDialog.java` — Removed the string separator constants and
  updated the view builders to use `LayoutParams.setMarginStart(8dp)`
  instead.

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