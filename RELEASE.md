# Llama Music Player v1.6.5

**Release date:** November 24, 2026

## Overview

Version 1.6.5 keeps the seek bar visible in a short undocked
player window. The song-info group and the seek-bar section
share the space between the top bar and the transport row
equally, instead of the seek-bar section being hidden there.
It is the same window the 1.6.4 entry described — short
portrait split-screen, and short, narrow landscape — with
the same evaluation, now used for sharing rather than for
hiding.

## Changed

- **The song-info group and the seek-bar section share the
  short undocked window.** In `MusicPlayer.applyAlbumArtSize`,
  `shareSpace` is `albumArtHidden && !dockSongInfo`, the
  same evaluation the 1.6.4 entry used for hiding. That
  value is passed to `applySongInfoLayout(albumArtHidden,
  dockSongInfo, shareSpace)`, the one writer for this
  layout. When `shareSpace` is true, `mSongInfoGroup` and
  `mSeekBarSection` each take 0dp height with weight 1,
  with an 8dp gap between them — `SONG_INFO_SEEK_BAR_GAP_DP`
  as the seek-bar section's top margin — and each keeps its
  content centred in its half: the group through its centred
  title/artist column, the seek-bar section through
  `Gravity.CENTER_VERTICAL`. The pair as a group fills the
  space between the top bar and the transport row, group on
  top and seek bar below it.
- **The new constant matches the existing shape.**
  `SONG_INFO_SEEK_BAR_GAP_DP = 8` sits with
  `SPLIT_SCREEN_FULL_HEIGHT_DP`, `SPLIT_SCREEN_MIN_HEIGHT_DP`,
  `DOCK_MIN_WIDTH_DP`, `ALBUM_ART_FULL_DP`,
  `ALBUM_ART_SHRUNK_DP`, and `TRANSPORT_ROW_SPLIT_BOTTOM_DP`:
  upper-snake case with a `_DP` suffix, one constant per
  layout fact. The same names carry across the layout,
  the activity, and these docs: `mSongInfoGroup`,
  `mSeekBarSection`, `mTransportRow`, `applySongInfoLayout`,
  `applyAlbumArtSize`, `setSongInfoDocked`.
- **Full-size and docked layouts are unchanged.** In a
  full-size window the group takes the leftover space with
  weight 1 and the seek-bar section hugs its content below
  it: `WRAP_CONTENT` height, weight 0, top margin 0,
  gravity `mDefaultSeekBarSectionGravity`. In a short, wide
  landscape window the group still docks beside the transport
  row through `setSongInfoDocked`: the two share the docked
  row with 0.5 weight each, and the seek-bar section takes
  what is left above it with weight 1. The dock condition
  is unchanged: art hidden, orientation landscape, and
  window width at least `DOCK_MIN_WIDTH_DP` (600dp).

## Preserved

- No change to the 1.6.4 resync: `mThemeRecreationPending`,
  `onLanguageChangedEvent`, `syncDisplayFromService`,
  `MusicService.requestPlayerResync()`,
  `MusicService.setActivityInForeground(boolean)`, and
  `UPDATE_PLAYER` are unchanged. `MusicService` stays at
  1.6.4.
- No change to the toast lane: 48dp
  (`TRANSPORT_ROW_SPLIT_BOTTOM_DP`) in split view and in
  landscape, plus the bottom buttons row's 16dp top padding.
  In portrait with the art visible the lane is the layout's
  64dp transport padding instead.
- No change to the 1.6.4 button widths: top-bar buttons stay
  floored at 84dp, header and language buttons stay floored
  at 68dp, with no padding change.
- No change to scope matching, repeat, shuffle, sort, search
  filtering, favorite persistence, or click debounce.
- No change to ProGuard rules. The 1.6.5 change adds no new
  subscriber and no new kept field.

For the full change history, see the [CHANGELOG](CHANGELOG.md). For
technical specifications, installation, and usage, see the [README](README.md).

Llama Music Player remains licensed under the MIT License.