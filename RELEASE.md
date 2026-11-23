# Llama Music Player v1.6.3

**Release date:** November 23, 2026

## Overview

Version 1.6.3 does one thing. It gives the top-bar and header buttons a
minimum width, so short labels in short languages no longer look too short
next to the dialog buttons. Top-bar buttons get 84dp, header and language
buttons get 76dp, with no padding change.

## Fixed

- **Top-bar buttons get a minimum width of 84dp.** The Cancel, Enable, Update,
  Modify, and Search buttons in Equalizer, Folder Manager, Metadata Editor,
  and Playlist are 36dp high with 14sp text and wrap_content width. Their
  width is now floored at 84dp, matching the dialog buttons, with no padding
  change.
- **Header buttons get a minimum width of 76dp.** The Sort button in Playlist
  and the Scope button in Folder Manager are 28dp high with 12sp text and
  wrap_content width. Their width is now floored at 76dp, with no padding
  change.
- **Language button gets a minimum width of 76dp.** The language button in the
  Credits dialog is 28dp high with 12sp text. It is built in code, so it now
  calls setMinWidth(dpToPx(76)) in createLanguageButton(), with no padding
  change. Short names such as EN no longer collapse to about 40dp.

## Preserved

- No change to the 1.6.2 fixes: cold start still restores paused to READY,
  dialog window context and locale lifecycle still stay fixed, and equalizer
  and playback parameters still stay preserved across audio-session rebuilds.
- No change to scope matching, repeat, shuffle, sort, search filtering,
  favorite persistence, or click debounce.
- No change to fill-screen buttons: Preset and Reset in Equalizer and Import
  Folder in Folder Manager stay match_parent, and the Music Player bottom
  buttons stay 0dp with weight. Their width comes from layout, not from
  minWidth.
- No change to icon buttons: the Music Player transport buttons and the
  Playlist Clear button stay at their fixed sizes with minWidth 0dp.
- No change to button padding, height, text size, or click behavior. Only
  minimum width changed, only for top-bar, header, and language buttons.
- No change to ProGuard rules. The 1.6.3 changes add no new subscriber and no
  new kept field.

For the full change history, see the [CHANGELOG](CHANGELOG.md). For
technical specifications, installation, and usage, see the [README](README.md).

Llama Music Player remains licensed under the MIT License.