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
- The idle transition clears current and target indices and tears down the
  playback notification, including when the service is already idle with
  nothing loaded. The existing state broadcast informs the UI of the empty
  state.

## Preserved

- The reset uses the existing `LIBRARY_RESET` handling path and existing
  stop/idle transition methods; no new methods were added.
- Both library-only and factory reset paths clear scope state because both
  reset the library.

For the complete release history, see [CHANGELOG.md](CHANGELOG.md).