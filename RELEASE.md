# Llama Music Player v1.6.7

**Release date:** November 26, 2026

## Overview

Version 1.6.7 sets the gap between the song-info group and the
seek-bar section to 12dp, declared once in
`MusicPlayer.SONG_INFO_SEEK_BAR_GAP_DP` and applied as the
seek-bar section's top margin. The layout XML, the Java constant,
and the player documentation all state the same 12dp value.
Playlist display updates and service playback are also kept in sync
across song transitions and playback-scope changes. The service
remains the authority for playback state, while the playlist screen
refreshes after the existing transition debounce and resolves the
current song again against the rebuilt display list. A song excluded
by a scope change may finish under the grace period, but it cannot
be selected for another play; Repeat One repeats it only if it
remains in the active scope.

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
- Playlist updates reject stale generations and use the existing
  500ms transition debounce. After rebuilding the display list, the
  current song path is resolved again before its row is highlighted
  or scrolled. A missing immediate lookup does not trigger an early
  rebuild.
- Scope changes capture successor paths from the old scoped playlist
  before rebuilding. If the current song is excluded, the first
  surviving successor is selected. If no old successor survives but
  the new scope contains songs, playback starts at the first song
  in the new scope. Playback goes idle only when the new scope is
  empty.

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
- Automatic completion, repeat-one, repeat-all, shuffle, user
  Next and Previous, and explicit song selection retain their
  existing service transition and history paths. Scope reconciliation
  does not introduce a parallel playback state machine.
- A song kept temporarily in the working playlist by the grace period is
  not eligible for another play after it leaves the active scope. Repeat One
  therefore does not restart an out-of-scope song; playback hands off at
  completion.

For the complete release history, see [CHANGELOG.md](CHANGELOG.md).