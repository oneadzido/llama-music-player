# Llama Music Player v1.6.7

**Release date:** November 26, 2026

## Overview

Version 1.6.7 sets the gap between the song-info group and the
seek-bar section to 12dp, declared once in
`MusicPlayer.SONG_INFO_SEEK_BAR_GAP_DP` and applied as the
seek-bar section's top margin. The layout XML, the Java constant,
and the player documentation all state the same 12dp value.

## Fixed

- The song-info group and the seek-bar section share the space
  between the top bar and the transport row equally: each takes
  0dp height with weight 1, and a 12dp gap separates them
  (`MusicPlayer.SONG_INFO_SEEK_BAR_GAP_DP`, applied as the
  seek-bar section's top margin). The seek-bar section wraps the
  SeekBar and the timer row in an inner container with centre
  gravity, so the pair reads as centred in the section's half
  regardless of the timer row's visibility.
- The behaviour holds in every undocked case: full-size portrait,
  short portrait split screen, and short, narrow landscape split
  screen. Undocking restores the seek-bar section's saved layout
  params, so the XML baseline, including the 12dp gap, returns
  with the section.

## Preserved

- All ten playlist sort options and existing playback controls.
- Equalizer sound processing, presets, and saved settings.
- Album-art loading and carousel transitions.
- Full-size album-art presentation and the side-by-side
  full-landscape layout.
- The established short-window layout and landscape docking
  behaviour.
- The timer row's fit rules: shown when it fits, hidden when it
  does not, with its top margin zeroed on the same turn it is
  hidden.

For the complete release history, see [CHANGELOG.md](CHANGELOG.md).