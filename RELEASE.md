# Llama Music Player v1.6.9

**Release date:** November 27, 2026

## Overview

Version 1.6.9 removes the playback notification and stops the service when the app task is dismissed while playback is not actively running.
A loaded song in PAUSED or READY state is saved for later restoration but no longer leaves a stale notification in the notification area.

## Fixed

- `MusicService.onTaskRemoved()` now keeps the service only when playback is actively running, including a `PREPARING` transition committed to start.
- When the task is removed in `PAUSED` or `READY`, the service clears the playback notification and stops itself instead of leaving an orphaned notification in the notification area.
- The full player state and player state are saved before teardown so the loaded song remains available for restoration when the app is opened again.

## Preserved

- Active background playback and its notification remain available when the app task is dismissed.
- The existing saved-state restoration path is retained; the task-removal handling does not discard the loaded song by forcing a library or playlist reset.

For the complete release history, see [CHANGELOG.md](CHANGELOG.md).