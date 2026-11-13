# Llama Music Player v1.4.8

**Release date:** November 13, 2026

Version 1.4.8 ships four changes. The Credits dialog's theme
picker no longer rebuilds the app when the user re-selects the
theme that is already applied. The equalizer's status header again
names the applied preset, correcting a regression introduced by a
rewrite of the equalizer activity. The Custom preset's restore
path in the equalizer is narrowed to the band levels it owns, and
the reload path is completed so a restart brings back the user's
full tuning. The preset-name fallback and its backing string are
removed. For the full change history, see the `[1.4.8]` entry in
the [CHANGELOG](CHANGELOG.md). For technical specifications,
installation, and usage, see the [README](README.md).

---

## What's in this release

### Re-selecting the current theme no longer rebuilds the app

`ThemeDialog` was the only picker that called its manager from
inside its own dismiss-delay runnable. Tapping any row — including
the row of the theme that was already applied — called
`ThemeManager.setThemeColor()`, which persists the value and
broadcasts `THEME_COLOR_CHANGED`. Every activity answered by
recreating, and every open dialog answered by dismissing.
Re-picking the theme you were already on closed the picker and
rebuilt the app behind it.

`ThemeDialog` now reports the tap and does not persist it. The
`CreditsDialog` listener owns the persist call, exactly as the
`LanguageDialog` listener has always owned
`LanguageManager.setLanguage()` and the `PresetDialog` listener
has always owned `EqualizerManager.applyPreset()`. The listener
already carried a guard against re-applying the current theme;
that guard is now the only gate on the reselection path, so it
takes effect. On reselection the picker still closes and the
tapped row still highlights and bounces — the tap is acknowledged
— but the persist, the broadcast, the listener's downstream work,
and the recreation are skipped.

### The equalizer's status header again names the applied preset

A rewrite of `EqualizerActivity` replaced the preset-aware status
line with a transient confirmation that was never meant to be the
resting state. The line read `Equalizer: Enabled` after any preset
was applied, disagreeing with the preset button beside it. It now
reads `Equalizer: <presetName>` — `Equalizer: Headphones`,
`Equalizer: Custom`, and so on — the state the line carried before
the rewrite. Every writer of the line now resolves through one
helper, `buildPresetStatusText()`, so the button and the status
line cannot disagree again.

### The Custom preset restores bands, and a restart restores everything

Selecting Custom from the preset picker restores the ten band
levels to the values the user last tuned by hand. Bass boost,
surround, pitch, and speed are independent of preset selection and
are not touched by the picker. The manager had been restoring bass
boost and surround on the picker path as well, so selecting Custom
after adjusting bass boost on a named preset silently reset bass
boost to the snapshot value. That restore is removed.

A restart still brings back the user's full tuning. The
equalizer's reload path now restores the custom pitch and speed
values as well as the custom bands and boosters, so the pitch and
speed sliders show the values the user tuned before the app was
closed. The snapshot is written when the user leaves Custom or the
session detaches, and read when the session attaches.

### The preset-name fallback is removed

`EqualizerManager.getPresetName(int)` was branching to a hardcoded
`Custom` name when the preset index was out of range or the
preset-name array could not be read. Neither condition is
reachable from a valid state, so the fallback was hiding broken
state rather than serving a legitimate one. The method now reads
the array directly and lets an out-of-range index fail loudly.
`loadPresetNames` no longer swallows a resource-read failure. The
`equalizer_preset_custom` string is removed from every locale;
`preset_names[0]` is the sole carrier of the word `Custom`.