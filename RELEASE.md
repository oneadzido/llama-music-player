# Llama Music Player v1.6.5

**Release date:** November 24, 2026

## Overview

Version 1.6.5 keeps the seek bar visible in a short undocked
player window. The song-info group and the seek-bar section
share the space between the top bar and the transport row
equally, instead of the seek-bar section being hidden there.
The timer row below the SeekBar shows only when the seek-bar
section can show it in full. It is the same window the 1.6.4
entry described — short portrait split-screen, and short,
narrow landscape — with the same evaluation, now used for
sharing rather than for hiding.

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
- **The timer row shows only when it fits in full.** The
  timer row (`mTimeRow`, `R.id.timeRow`) holds the
  current time and the total duration below the SeekBar. In
  a full-size window it always shows: the seek-bar section
  hugs its content. In a short window where the section is
  given a fixed share, `MusicPlayer.updateTimeVisibility`
  compares the section height with what the row needs: the
  SeekBar height, the row's 8dp top margin, the row's height,
  and the section's top and bottom padding. When the section
  is shorter, the row is hidden (`GONE`) and the SeekBar
  stays, centred in its share. When space returns, the row
  returns. The check runs after `applySongInfoLayout` and
  again whenever the section's height changes, so dragging
  the split-screen divider re-evaluates it. There is no
  orientation rule and no hard-coded height threshold: the
  decision is a measurement of what fits.
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
- **Full-size layout is unchanged; the docked layout keeps
  its shape and follows the same timer rule.** In a
  full-size window the group takes the leftover space with
  weight 1 and the seek-bar section hugs its content below
  it: `WRAP_CONTENT` height, weight 0, top margin 0,
  gravity `mDefaultSeekBarSectionGravity`, and the timer row
  always shows. In a short, wide landscape window the group
  still docks beside the transport row through
  `setSongInfoDocked`: the two share the docked row with 0.5
  weight each, and the seek-bar section takes what is left
  above it with weight 1. The dock condition is unchanged:
  art hidden, orientation landscape, and window width at
  least `DOCK_MIN_WIDTH_DP` (600dp). Because the docked
  seek-bar section is also given a fixed share, its timer
  row shows only when it fits in full, the same rule as in
  the shared layout.

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
  `BroadcastTranslator` subscriber and no new kept field. The
  timer check is a layout-change listener on the seek-bar
  section, and `mTimeRow` is a view reference.

For the full change history, see the [CHANGELOG](CHANGELOG.md). For
technical specifications, installation, and usage, see the [README](README.md).

Llama Music Player remains licensed under the MIT License.