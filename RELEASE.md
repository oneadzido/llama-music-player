# Llama Music Player v1.5.3

**Release date:** November 17, 2026

Version 1.5.3 improves dialog reliability, activity recreation handling,
and music-player interface responsiveness. It fixes an issue where
the Credits dialog's theme and language buttons could become
unusable after opening and closing their pickers without making a
selection. The pickers now notify the host through a dedicated
dismiss callback, allowing the Credits dialog to clear its picker-open
flags independently of selection handling.

This release also improves coordination between dialog dismissal and
activity recreation during language and theme changes.
Playback and favorite controls now explicitly refresh their visual
states after toggle actions, addressing cases where the underlying
action succeeded but the corresponding icon remained visually
unchanged.

For the full change history, see the `[1.5.3]` entry in the
[CHANGELOG](CHANGELOG.md). For technical specifications,
installation, and usage, see the [README](README.md).

---

## What's in this release

### The Credits dialog buttons remain usable after closing the
pickers

The Credits dialog tracks whether the theme and language pickers
are showing to prevent multiple instances from opening. Previously,
closing either picker without selecting an item could leave its
corresponding flag set, preventing the button from opening the
picker again.

Both pickers now provide a dedicated dismissal callback to the host.
This callback is independent of the selection callback and runs
whenever the picker is dismissed. The Credits dialog can therefore
clear its picker-open flag regardless of whether the user selects
an item, reselects the current item, or closes the picker without
making a selection.

### Language and theme changes have improved recreation
handling

Activity recreation is now guarded against duplicate requests and
is posted through the activity's window decor view. The recreation
request checks the activity's lifecycle state before proceeding,
improving coordination between UI teardown and activity
recreation.

The Credits dialog also dismisses directly when responding to
language and theme change events rather than scheduling an
additional dismissal runnable. This reduces competing dismissal
operations during the recreation process.

### Playback and favorite icons reflect toggle actions

Play/pause and favorite controls now explicitly call
`refreshTransportControls()` after their respective toggle
operations. This allows the existing transport-control renderer
to retrieve the current service state and update the corresponding
icons without relying on manual activity recreation.

### Existing playback controls are preserved

The existing repeat and shuffle behavior remains intact. The
changes use the existing transport-control refresh mechanism
rather than introducing a separate rendering system.