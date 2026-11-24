# Llama Music Player v1.6.4

**Release date:** November 24, 2026

## Overview

Version 1.6.4 makes the player, equalizer, and editor update
instantly instead of waiting for a recreate or the next playback
state save, and makes long text truncate instead of scrolling
away. It fixes the case, seen on TECNO / HiOS, where equalizer
pitch and speed, the carousel, and the song info only refreshed
after an activity recreate. It also tightens the header and
language buttons from 76dp to 68dp.

## Fixed

- **Equalizer pitch and speed now reach the service whether or not
  it is playing.** `EqualizerActivity` dropped the `isPlaying()` gates
  (pitch, speed, and reset: five sites). A drag made while paused, in
  `READY`, or in `PREPARING` used to write the preference and never
  tell the service, so nothing changed until the next playback state
  save. The service already applies parameters in `READY` and before
  `start()`; now the equalizer actually sends them.
- **The player resyncs when it returns to the foreground.** The
  service re-broadcasts the full playback snapshot when the player
  goes from background to foreground, with `requestPlayerResync()` as
  a backup for a rebind where the flag did not flip, and the player
  pulls a full snapshot on bind and on resume. The carousel, song
  info, transport controls, and seek bar refresh without a recreate.
  `UPDATE_PLAYER` is posted, not sticky, so a player that was
  destroyed or backgrounded while a change broadcast fired used to
  miss it entirely.
- **Language recreate is deferred in `MusicPlayer`, like theme.**
  `onLanguageChangedEvent` parks the recreate in
  `mThemeRecreationPending` when the activity is not started, and
  `onStart()` runs it on return. Recreating a backgrounded activity
  can be deferred or dropped by aggressive OEMs, and the recreate can
  race the picker's teardown. Both actions are just a recreate, so
  they share one pending flag. Other activities still recreate
  immediately on a language change.
- **The Playlist search hint truncates with an ellipsis.** The hint
  only ellipsizes while horizontal scrolling is off, so the field now
  toggles: empty means hint showing, ellipsize on, scrolling off;
  typing means ellipsize off, scrolling on, so a long query scrolls
  instead of being cut off.
- **Metadata fields truncate in view mode and scroll in edit mode.**
  `setFieldsEnabled` now calls `applyFieldTruncation` for every field.
  In view mode the field truncates with "...": ellipsize on,
  horizontal scrolling off, the parent scroll wrapper disabled, and
  the field capped to the wrapper's measured width, which is what
  makes the ellipsis actually appear inside the wrapper's
  unspecified-width measure pass. In edit mode the field scrolls
  again. File info is never edited, so it is always truncated.
- **The seek bar in a short portrait window hides only when the layout
  genuinely cannot fit it.** A short, undocked window cannot fit
  header + group + seek bar + transport + bottom buttons, so the seek
  bar is the row that goes; the group stays, because it is the
  glanceable part. Everywhere else the seek bar stays visible. The
  group stays vertically centered in the space between the header and
  whatever follows it, with no extra top padding. The SeekBar keeps
  receiving progress updates while hidden, so returning to a
  full-size window shows the current position with no re-sync.
- **Header and language buttons move from 76dp to 68dp.** The Sort
  button in Playlist, the Scope button in Folder Manager, and the
  language button in the Credits dialog are 28dp high with 12sp text.
  Their minimum width drops from 76dp to 68dp, with no padding
  change. 76dp left too much empty width on buttons only 28dp high;
  68dp keeps short labels from collapsing without the extra width.

## Preserved

- No change to the 1.6.3 top-bar buttons: they stay floored at 84dp,
  with no padding change.
- No change to the 1.6.2 fixes: cold start still restores paused to
  READY, dialog window context and locale lifecycle still stay fixed,
  and equalizer and playback parameters still stay preserved across
  audio-session rebuilds.
- No change to scope matching, repeat, shuffle, sort, search filtering,
  favorite persistence, or click debounce.
- No change to fill-screen buttons: Preset and Reset in Equalizer and
  Import Folder in Folder Manager stay match_parent, and the Music
  Player bottom buttons stay 0dp with weight. Their width comes from
  layout, not from minWidth.
- No change to icon buttons: the Music Player transport buttons and the
  Playlist Clear button stay at their fixed sizes with minWidth 0dp.
- No change to button padding, height, text size, or click behavior.
- No change to ProGuard rules. The 1.6.4 changes add no new subscriber
  and no new kept field.

For the full change history, see the [CHANGELOG](CHANGELOG.md). For
technical specifications, installation, and usage, see the [README](README.md).

Llama Music Player remains licensed under the MIT License.