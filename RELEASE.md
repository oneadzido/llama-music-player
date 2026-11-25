# Llama Music Player v1.6.6

**Release date:** November 25, 2026

## Overview

Version 1.6.6 improves playlist sorting and keeps the Music Player and
Equalizer screens responsive to playback changes. The player layout also
adapts to resized windows, showing the timer only when it fits and
keeping the seek bar, title, and artist legible and centred in every
undocked case.

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
- The SeekBar and timer row sit centred in their half of the player
  layout instead of being pinned to the top, in every undocked case:
  full-size portrait, short portrait split screen, and short, narrow
  landscape split screen. The stacked layout declares its share model
  in the XML: the song-info group and the seek-bar section each take 0dp
  height with weight 1, a 16dp gap separates them
  (`SONG_INFO_SEEK_BAR_GAP_DP` as the seek-bar section's top margin),
  and the seek-bar section holds the SeekBar and timer row in an inner
  container with centre gravity.
- The timer row hides in short split-screen landscape windows where it
  would otherwise be clipped by the bottom of the screen. The fit check
  compares the predicted timer bottom against the actual visible
  activity window bounds, not only against the seek-bar section's own
  height. The timer is shown in full-size landscape and portrait, and
  hidden in short split-screen landscape.
- The timer row's top margin is zeroed on the same turn the row is
  hidden, so a hidden timer leaves no empty gap below the SeekBar and
  the SeekBar stays centred in its share.
- The title and artist stay legible and vertically clear of the container
  edges in a short shared half. The song-info group holds an 80dp minimum
  height, so the title, artist, and the 48dp Home/Favorite buttons stay
  fully visible with clear space above and below in every undocked case.

## Preserved

- All ten playlist sort options and existing playback controls.
- Equalizer sound processing, presets, and saved settings.
- Album-art loading and carousel transitions.
- Full-size album-art presentation and the side-by-side full-landscape layout.
- The established short-window layout and landscape docking behaviour.

For the complete release history, see [CHANGELOG.md](CHANGELOG.md).