# Llama Music Player v1.6.8

**Release date:** November 27, 2026

## Overview

Version 1.6.8 makes a full library reset clear playback-scope preferences
and in-memory scope state together with the library, preventing stale scope
data from surviving into a newly selected scope.

## Fixed

- Full library reset removes `playback_scope_json`,
  `playback_scope_resume_path`, and `playback_scope_empty_state`.
- The service clears its full, scoped, and current playlists, resets the
  scope to `ALL_SONGS`, and empties `mScopeHandoffCandidates` before
  transitioning playback to idle.
- Before scope and library state are cleared, the service invalidates pending
  playback work, stops and releases the active player on its owning thread,
  and removes the playback foreground notification.
- Notification teardown explicitly ends foreground-service status and cancels
  the playback notification. The service then clears scope state and performs
  the idle transition, whose state broadcast informs the UI of the empty state.
- The transition-generation barrier rejects queued updates from the song that
  was playing before reset, preventing its notification from reappearing.
- The Music Player applies a complete no-song timing state after reset:
  the disabled SeekBar returns to zero, and elapsed time and duration return
  to `00:00`. The periodic time updater does not repaint stale timing while
  the service playlist is empty.

## Preserved

- The reset uses the existing `LIBRARY_RESET` handling path and existing
  stop/idle transition methods; no new methods were added.
- Both library-only and factory reset paths clear scope state because both
  reset the library.

For the complete release history, see [CHANGELOG.md](CHANGELOG.md).