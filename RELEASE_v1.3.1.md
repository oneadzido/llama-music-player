# Llama Music Player v1.3.1

**Release date:** Oct 13, 2026

## Overview

Two corrections to the language-change flow and one layout correction
in the theme dialog.

The language-change flow is simplified. The dialogs no longer carry
their own `LANGUAGE_CHANGED` receivers, and `MusicPlayer` no longer
defers its recreation until a dialog closes. A language change now
closes every open dialog and recreates the activity on the same turn
the broadcast is delivered. The next time a dialog is opened it is
constructed against the new locale from the start, so its strings and
its layout direction are correct on the first frame it renders.

The theme dialog's row carries the same shape as the preset dialog's
row. The row's own padding is symmetric and logical, and each child
that sits inside the row carries at most a single logical start
margin. The trailing inset on whichever child lands at the trailing
edge is supplied by the row's own end padding, which is what keeps the
row correct under LTR and RTL without any special-casing.

## Fixed

- **Language changes close the surface the choice was made from and
  recreate the activity in one step.** The picker, the Credits
  dialog, and the activity all reacted to the same
  `LANGUAGE_CHANGED` broadcast in different ways. The picker
  dismissed itself. The Credits dialog rebuilt its own content view
  in place. The activity deferred its recreation until the Credits
  dialog closed. The result was that the surface the user was on
  survived the choice, and it was rendered against the new locale by
  an in-place rebuild that could not pin every child's layout
  direction before the frame in which the rebuild was installed.
  The row whose `START`/`END` gravity resolved against the outgoing
  direction first was the one that showed the misbehaviour — the
  theme dialog's current row, whose indicator landed on the wrong
  physical side until the next lifecycle event recreated the dialog
  and corrected it.

  The flow is now a single step. The dialogs dismiss themselves on
  the broadcast and no longer attempt to rebuild their content view.
  `MusicPlayer` recreates immediately on the same turn. Every surface
  the user was on when the choice was made closes as part of the
  choice, and the activity is rebuilt with a wrapped base context
  that carries the new locale. The next time a dialog is opened, it
  is constructed against the new locale, and every string, every
  `START`/`END` gravity, and every weighted row resolves against the
  correct direction from the first frame.

- **Theme rows carry symmetric edge insets in both directions.**
  The row's own padding was an asymmetric pair: sixteen dp on the
  physical left, zero on the physical right. Under LTR the left is
  the leading edge, so the name was inset and the trailing side was
  spaced by the margins on the colour circle and the indicator.
  Under RTL the two physical sides swap roles, and the row's leading
  edge — now the right — carried no inset at all. The theme name sat
  against the row's right edge for that reason. The row's padding is
  now symmetric and logical: `setPaddingRelative` with the same
  `LIST_ITEM_PADDING_HORIZONTAL_DP` on both sides, so the leading
  and trailing edges carry the same inset in every locale.

- **The colour preview circle carries a single logical start margin.**
  The circle had two margins, a start margin of eight dp and an end
  margin of sixteen dp. Under RTL the two stacked with the
  indicator's own margins on the same physical side of the row, so
  the circle appeared to have roughly twice the space it was designed
  to have on the side it sat closest to. The circle now carries only
  its start margin — the gap between the name and the circle — and
  lets the row's own end padding supply the trailing inset for
  whichever child lands there. `COLOR_CIRCLE_MARGIN_END_DP` is
  removed; the constant had no reader after the margin went away.

- **The `CURRENT` indicator carries a single logical start margin.**
  The indicator had a start margin of eight dp and an end margin of
  sixteen dp. The end margin was compensating for the row's missing
  trailing padding, and it stacked with the circle's own end margin
  under RTL in the same way the circle's did. The indicator now
  carries only its start margin — the gap between the circle and the
  indicator — and the row's own end padding supplies the trailing
  inset on the side the indicator sits on.
  `LIST_ITEM_PADDING_HORIZONTAL_DP` is the single writer for that
  edge.

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

- `ThemeDialog.COLOR_CIRCLE_MARGIN_END_DP` is removed.

- `ThemeDialog`, `PresetDialog`, and `LanguageDialog` each lose their
  `LANGUAGE_CHANGED` receiver, its registration, its unregistration,
  its field, and the `BroadcastReceiver`, `Intent`, `IntentFilter`,
  `ContextCompat`, and `Configuration` imports that had no other
  reader. Each also loses its `rebuildForLocale()` and its
  `pinContentLayoutDirection(View)` helper. `ThemeDialog` and
  `PresetDialog` also lose their `dismiss()` override, because the
  only thing it did was unregister the language receiver.

- `CreditsDialog` loses `rebuildForLocale()` and
  `pinContentLayoutDirection(View)`. Its `LANGUAGE_CHANGED` receiver
  now calls `dismiss()` instead of rebuilding. Its theme receiver is
  unaffected; the theme and the language are separate concerns.

- `MusicPlayer.mLanguageChangedReceiver` calls `recreate()` directly
  on the broadcast's turn. The `mPendingLanguageRecreate` field and
  the `consumePendingLanguageRecreate()` method are removed, along
  with the calls to the latter from `onResume()` and
  `onWindowFocusChanged(boolean)`.

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