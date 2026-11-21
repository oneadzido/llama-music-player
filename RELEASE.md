# Llama Music Player v1.5.9

**Release date:** November 21, 2026

Version 1.5.9 fixes three things found after 1.5.8 shipped: a
cold-start auto-start that left the transport controls stale, a
narrow landscape window that cramped the docked artist group
and transport row, and an extra top gap above the artist group
in the stacked portrait layout. It also renames the seek-bar
progress drawable to say what it is: the enabled-state
progress, `seekbar_progress_enabled`.

Llama Music Player itself remains licensed under the MIT License.

For the full change history, see the `[1.5.9]` entry in the
[CHANGELOG](CHANGELOG.md). For technical specifications, installation,
and usage, see the [README](README.md).

---

## What's in this release

### Fixed: cold-start auto-start now commits the UI

When the app was playing before it exited, a cold start
restores the playlist and resumes playback. The service
committed nothing after the reconcile on that path, so the
transport controls kept the pre-restore state until the next
playback change forced a refresh.

The cold-start branch in `MusicService.setPlaylist` now
calls `broadcastUpdate()` after `reconcile()` when it is a
cold start and the song is resuming, so every surface
commits on the same turn the auto-start lands.

When the app was paused before it exited,
`mPauseAfterPrepare` stays true and no commit fires: the
player stays paused, as before.

### Fixed: narrow landscape uses the stacked portrait layout; short, undocked windows hide the seek bar

Docking splits the window 0.5 / 0.5. In landscape
split-screen the window is about 400dp wide, so the
transport row got about 200dp against the 288dp it needs
and the artist group kept its 72dp + 72dp inner padding,
leaving about 56dp for the title: the title truncated to
a fragment and the transport buttons squeezed together.

The dock now has a width gate: it applies only when the
window is at least 600dp wide (`DOCK_MIN_WIDTH_DP`), on
top of the existing art-hidden and landscape conditions.
A narrow landscape window falls back to the stacked
portrait layout instead, group on its own row and
transport on its own row, each at full width — but a
short, undocked window cannot fit the whole stacked
layout either, so the seek-bar section is hidden there.
That is the same evaluation in short portrait
split-screen and in short, narrow landscape:
`albumArtHidden && !dockSongInfo`. The
choice is deliberate: the group is the glanceable part —
what is playing, home, favourite — plus the transport
controls, while scrubbing can wait until the window is
full-size again. Hiding the group instead would leave
controls for a song the user cannot identify, and would
take home and favourite with it, since both live inside
the group. The SeekBar keeps receiving progress updates
while hidden, so returning to a full-size window shows
the current position with no re-sync. Full landscape, at
800dp and wider, still docks as before, with the seek
bar visible.

### Fixed: artist group stays centered, without the extra top gap

The stacked layout used to pin the group to the top when
the art was hidden and add the album-art wrapper's 16dp
top padding. That pushed the group down from where
centering would put it and read as an extra gap at the
top.

The pin-to-top is removed. The group stays vertically
centered in the space between the header and the seek
bar, with its default top padding, in portrait and in a
narrow landscape window that falls back to the stacked
layout. The docked layout is unchanged: the docked row
still hugs its content and the toast lane is still the
row's 48dp bottom margin.

### Changed: seek-bar progress drawable renamed

`seekbar_progress` is now `seekbar_progress_enabled`.
The references in `activity_music_player.xml` and
`EqualizerActivity` are updated to the new name. There is
no visual change: the drawable itself is untouched, only
its name says which state it draws.

### Existing behavior preserved

No change to scope matching, repeat, shuffle, sort, search
filtering, favorite persistence, or click debounce. No
change to the theme and language dialog behaviour from
1.5.8, or to the notification commit rule: one commit per
transition, through `broadcastUpdate()`. No change to
ProGuard rules: the 1.5.9 changes add no new subscriber
and no new kept field.