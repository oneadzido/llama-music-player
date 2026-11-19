# Llama Music Player v1.5.6

**Release date:** November 19, 2026

Version 1.5.6 fixes a stale-target end-of-song deadlock after a
playback rebuild while playing, and makes the playlist re-evaluate
its displayed list when a song is tapped.

Llama Music Player itself remains licensed under the MIT License.

For the full change history, see the `[1.5.6]` entry in the
[CHANGELOG](CHANGELOG.md). For technical specifications, installation,
and usage, see the [README](README.md).

---

## What's in this release

### Fixed: stale-target end-of-song deadlock

After a scope, favorite, sort, or `setPlaylist()` rebuild while
`PLAYING`, a stale `mTargetIndex` could suppress `onCompletion()`,
leaving playback stuck silent at end-of-song in the `PLAYING`
state. The health check could not save it, because the position
was already at the duration.

A handoff is now only real while `PREPARING`. While `PLAYING`, the
target must equal the current song, and `onCompletion()` never
suppresses completion for a stale target. `onCompletion()` aligns
a stale target to the current song before resolving the next song,
`applyPlaybackScopeInternal()` and
`reconcilePlaybackAfterScopeChange()` align a stale target while
`PLAYING`, and `onCompletion()` uses the new `scopeEmpty()` helper
to enter the no-song state when the scope has no eligible songs.

### Fixed: playlist refresh on song tap

`PlaylistActivity` now refreshes its displayed list on a song tap,
before highlighting the tapped row. The refresh re-reads the
service's current playlist after the service's `rebuildPlaylists()`
on commit, so it evaluates whether each song still belongs to the
current displayed list.

Songs no longer in scope and no longer current/target are evicted.
The tapped song survives as the grace-period current/target when it
committed, even if it is out of scope. A tapped song that never
became current is not highlighted as playing.
`onUpdatePlayerEvent()` refreshes again when the playing song is
not in the displayed list, covering a prepare that commits after
the tap's own refresh. No new service evaluation logic was added
for this refresh; the mechanism already existed in the service.

### Existing behavior preserved

Scope matching, repeat, shuffle, sort, search, favorite
persistence, and click debounce behavior are unchanged. Playlist
refresh remains suppressed during genre loading and user scrolling,
as before. No event delivery, dependency version, dialog lifecycle,
or animation changes were made.