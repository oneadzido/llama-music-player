# Llama Music Player v1.6.2

**Release date:** November 23, 2026

## Overview

Version 1.6.2 does three things, in this order. First, it makes cold start
always restore paused: the last track and position are prepared into `READY`
without autoplay, regardless of the previous session's playing flag. Second,
it fixes the shared dialog window context and the locale lifecycle: all five
dialogs get the original Activity context for their window and the
locale-wrapped context for their views, and the selected locale and timer
digits are reapplied after resume and configuration changes. Third, it
preserves the equalizer and playback parameters across audio-session rebuilds:
the attach snapshots and restores the full equalizer state, and playback
parameters are applied before start and in `READY`.

## Fixed

- **Cold app launch always restores paused.** The normal activity
  playlist-load path no longer consults the previous session's
  `wasPlayingBeforeRestart` flag to decide whether to autoplay. A cold launch
  prepares the last track/position and enters `READY`; only explicit
  pending-song launches are allowed to autoplay.
- **Dialog window context.** All five custom dialogs pass the original
  Activity context to the `Dialog` window, while dialog views are built from
  the locale-wrapped context. Dialog contexts are wrapped in the app locale
  before view creation. This corrects the shared superclass-context pattern
  implicated in the reported app-wide dialog crashes.
- **Locale and timer digits.** The selected app locale is reapplied after
  activity resume and configuration changes; timer digits use an explicit
  locale formatter for the selected locale.
- **Equalizer preserved on attach.** `EqualizerManager.attachToAudioSession()`
  snapshots and restores the current equalizer preset, band levels, bass
  boost, surround, pitch, and speed when attaching effects to a new audio
  session.
- **Playback parameters.** Playback parameters are applied on the player
  thread before auto-start and when the prepared player lands in `READY`.

## Preserved

- No change to the 1.6.1 fixes: playback still does not auto-resume when it
  should stay paused, and a sticky restart still re-seeds the playlist behind
  the notification and prepares the last song to `READY` without starting
  playback.
- No change to scope matching, repeat, shuffle, sort, search
  filtering, favorite persistence, or click debounce.
- No change to ProGuard rules. The 1.6.2 changes add no new subscriber and no
  new kept field.

For the full change history, see the [CHANGELOG](CHANGELOG.md). For
technical specifications, installation, and usage, see the [README](README.md).

Llama Music Player remains licensed under the MIT License.