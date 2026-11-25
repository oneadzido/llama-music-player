# Llama Music Player v1.6.6

**Release date:** November 25, 2026

## Overview

Version 1.6.6 improves playlist sorting and keeps the Music Player and
Equalizer screens responsive to playback changes. The player layout also
adapts to resized windows, showing the timer only when it fits.

## Fixed

- Playlist sorting now consistently follows the selected order during
  playback, including when the active playlist is refreshed.
- The current and next tracks remain correctly selected after a sort.
- Song title, artist, and playback position update without waiting for
  album-art transitions to finish.
- Equalizer controls refresh their displayed values promptly when playback
  settings change or the screen reconnects to playback.
- The timer row no longer gets cut off in a short player window. It hides
  when there is insufficient space and returns when the window grows.
- The SeekBar and timer row now sit centred in their half of the player
  layout instead of being pinned to the top, in every undocked case:
  full-size portrait, short portrait split screen, and short, narrow
  landscape split screen.

## Preserved

- All ten playlist sort options and existing playback controls.
- Equalizer sound processing, presets, and saved settings.
- Album-art loading and carousel transitions.
- Full-size album-art presentation and the side-by-side full-landscape layout.
- The established short-window layout and landscape docking behaviour.

For the complete release history, see [CHANGELOG.md](CHANGELOG.md).