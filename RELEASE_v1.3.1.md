# Llama Music Player v1.3.1

**Release date:** Oct 13, 2026

## Overview

Four layout defects in the theme dialog, all of them direction-related
and all of them visible under an RTL locale. The row's own padding was
an asymmetric pair of physical edges, so the Arabic screen put the
theme name against the leading edge with nothing between them. The
colour preview circle and the `CURRENT` indicator each carried their
own trailing margin on top of the row's trailing padding, so the
circle appeared to have roughly twice the space it was designed to
have on the side it sat closest to. The four corrections are the
whole of this release: the row's padding is now symmetric and logical,
the circle and the indicator each carry a single logical start
margin, and the constant the circle's trailing margin used is removed.

## Fixed

- **Theme rows carry symmetric edge insets in both directions** —
  The theme row's own padding was an asymmetric pair: sixteen dp on
  the physical left, zero on the physical right. Under LTR the left
  is the leading edge, so the name was inset and the trailing side
  was spaced by the margins on the colour circle and the indicator.
  Under RTL the two physical sides swap roles, and the row's leading
  edge — now the right — carried no inset at all, because the value
  the code wrote there was zero. The Arabic screen showed the theme
  name against the row's right edge for that reason. The row's
  padding is now symmetric and logical: `setPaddingRelative` with
  the same `LIST_ITEM_PADDING_HORIZONTAL_DP` on both sides, so the
  leading and trailing edges carry the same inset in every locale.

- **The colour preview circle no longer carries a doubled inset** —
  The circle had two margins, a start margin of eight dp and an end
  margin of sixteen dp. The row's trailing inset was supposed to be
  supplied by the row's own end padding, which the previous
  asymmetric pair did not set. Under RTL the circle's end margin
  and the indicator's own end margin both landed on the same
  physical side of the circle and stacked, so the circle appeared
  to have roughly twice the space it was designed to have on the
  side it sat closest to. The circle now carries only its start
  margin — the gap between the name and the circle — and the row's
  own end padding supplies the trailing inset for whichever child
  lands there. `COLOR_CIRCLE_MARGIN_END_DP` is removed; the
  constant had no reader after the margin went away.

- **The `CURRENT` indicator carries a single leading inset and no
  trailing margin** — The indicator had a start margin of eight dp
  and an end margin of sixteen dp. The end margin was compensating
  for the row's missing trailing padding, and it stacked with the
  circle's own end margin under RTL in the same way the circle's
  did. The indicator now carries only its start margin — the gap
  between the circle and the indicator — and the row's own end
  padding supplies the trailing inset on the side the indicator
  sits on. `LIST_ITEM_PADDING_HORIZONTAL_DP` is the single writer
  for that edge.

- **The two child views and the row agree on who supplies the
  trailing inset** — The row's padding, the circle's margins, and
  the indicator's margins were three separate statements about the
  same physical edge, and only one of them was supposed to be the
  writer. The row is now the writer for both edges, and each child
  carries one margin describing only the gap between it and its
  sibling. The same shape the preset dialog's row already uses.

## Under the Hood

- `ThemeDialog.createThemeItem()` calls `setPaddingRelative` with a
  symmetric pair of horizontal insets — `LIST_ITEM_PADDING_HORIZONTAL_DP`
  on both sides — instead of `setPadding` with a
  `LIST_ITEM_PADDING_HORIZONTAL_DP` value on the left and zero on
  the right.

- `ThemeDialog.createColorCircle()` sets only a start margin on the
  circle's `LayoutParams`. The end margin is removed.

- `ThemeDialog.createThemeItem()` sets only a start margin on the
  indicator's `LayoutParams`. The end margin is removed. The row's
  own end padding is the single writer for the trailing inset.

- `ThemeDialog.COLOR_CIRCLE_MARGIN_END_DP` is removed. The constant
  had no reader after the circle's end margin was removed.

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