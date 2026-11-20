# Llama Music Player v1.5.7

**Release date:** November 20, 2026

Version 1.5.7 focuses the current song after hiding search, and
preserves the equalizer pitch and speed across track transitions.

Llama Music Player itself remains licensed under the MIT License.

For the full change history, see the `[1.5.7]` entry in the
[CHANGELOG](CHANGELOG.md). For technical specifications, installation,
and usage, see the [README](README.md).

---

## What's in this release

### Fixed: current song focus after hiding search

Hiding search restored the full, unfiltered list but never scrolled
to the current song, so the list stayed wherever the filtered
results had left it.

`hideSearch()` now posts the same delayed, guarded
`focusCurrentSong()` the sort path uses, after
`performSearch("")` restores the full list. The delay lets the
adapter lay out the restored list first, and the guards suppress
the scroll while the user is scrolling, tapping, or genre-loading.
Both callers, the Cancel button and `onBackPressed()`, go through
`hideSearch()`.

### Fixed: pitch and speed preserved across transitions

Every transition creates a new `MediaPlayer`, which starts with
default `PlaybackParams` (1.00x speed, 0 semitones). Pitch and
speed are user preferences, not per-track state, so they are now
carried across the session change instead of resetting.

`MusicService` `onPrepared()` attaches through the new
`attachEqualizerPreservingPlaybackParams()` helper, which
snapshots pitch and speed before `attachToAudioSession()` and
restores them when the attach changed them. The READY branch
(cold-start restore, pause-during-prepare) now calls
`updatePlaybackParams()` instead of returning with default params
until a later resume. `EqualizerActivity` attaches through the
new `attachPreservingPitchSpeed()` helper at all three attach
sites (`onCreate()`, `onResume()`,
`onAudioSessionChangedEvent()`), so the refresh that follows a
transition reads the preserved values into the SeekBars.

### Existing behavior preserved

Search filtering, sort, scope matching, repeat, shuffle,
favorite persistence, and click debounce behavior are unchanged.
The intentional Reset button still returns pitch and speed to
defaults, and does not go through attach. No event delivery,
dependency version, dialog lifecycle, or animation changes were
made.