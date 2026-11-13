# Llama Music Player v1.4.7

**Release date:** November 13, 2026

Version 1.4.7 corrects a visual regression introduced in 1.4.6. The
`ABOUT LLAMA` title in the Credits dialog is now rendered in the
same static light grey that every other dialog uses for its top-bar
title, matching `PresetDialog`, `ThemeDialog`, `LanguageDialog`, and
`PlaybackScopeDialog` in every theme. For the full change history,
see the `[1.4.7]` entry in the [CHANGELOG](CHANGELOG.md). For
technical specifications, installation, and usage, see the
[README](README.md).

---

## What's in this release

### The Credits dialog's title matches the other dialogs

During the 1.4.6 theme rebuild, the Credits dialog's `ABOUT LLAMA`
title was inadvertently rewritten to use the active theme color,
while every other dialog's top-bar title stayed in the static light
grey (`#C0C0C0`). The title now reads from that same
`TEXT_COLOR_LIGHT` value, so the Credits dialog's top bar looks the
same as the rest of the app's dialog chrome.

The theme color continues to be applied to the buttons, the app
name, the library links, and the header accents. The change is
scoped to the title text alone.