# Llama Music Player v1.4.2

**Release date:** November 7, 2026

Version 1.4.2 refines the playback-scope dialog so an empty filter
selection is presented and applied for what it is. For the full change
history of this release, see the `[1.4.2]` entry in the
[CHANGELOG](CHANGELOG.md). For technical specifications, installation,
and usage, see the [README](README.md).

---

## What's in this release

Llama Music Player combines a scope-aware playback engine, persistent
media controls, and a compact interface designed to remain consistent
across all sixteen supported languages.

### Playback scope

The playback scope can select favorites, loose tracks, or specific
folders, genres, artists, and albums. Its live count uses the same
matching rules as the playback service.

A filter mode with no values checked is a scope that matches nothing.
The dialog shows that selection as it is: the mode stays selected, its
filter list stays expanded, and the header count reads zero. The
selection is not rewritten into All Songs; that scope is reached
through the All Songs row. The playing song and a pending next song
remain available while a scope change takes effect, allowing the
active track to finish naturally.

APPLY is enabled when the selection differs from the scope the dialog
opened with, and disabled otherwise. An empty filter selection from a
different starting scope is therefore a change the user can commit; a
selection identical to the one already applied is not. A preference
change made after APPLY is tapped cancels the queued commit and
restores the button to the state the current selection warrants.

At song completion, the service chooses the next track from the active
scope. Repeat off advances through the scoped songs and pauses when
the scope is empty or the current song is its final entry. Repeat all
wraps within the scope, while shuffle selects from the same scoped
collection. Repeat one continues the current song.

### Playback notification

The playback notification is rendered from the service's committed
playback snapshot and remains synchronized with the media session.
Notification rendering and foreground-service promotion are handled
as distinct operations, so the notification can remain visible when
Android denies a foreground promotion because notification permission
is unavailable.

A sticky service restart rebuilds the notification from persisted
playback state. During a prepare or player error recovery, the service
keeps transition state visible to the notification controller until
reconciliation completes. Failed renders remain eligible for another
synchronization attempt, and the media session retains the same
committed snapshot as the notification.

### Player and dialog interface

The player uses a Home control for returning to the Android home
screen. The two-tap confirmation leaves playback running in the
background and uses a contrasting border and glyph color in both
light and dark themes.

The Home and Favorite controls use the standard 48dp control-button
treatment. The transport row follows a 48dp / 52dp / 56dp size
hierarchy with 28sp / 32sp / 36sp glyphs. The playlist favorite control
uses a 36dp surface and a 20sp glyph.

Dialog and activity title bars use logical start and end margins for
their 8dp title gutters. The same layout rule applies to the Playback
Scope, Language, Credits, Preset, and Theme dialogs and to the Player,
Playlist, Folder Manager, Metadata Editor, and Equalizer activities.
This keeps the title centered between its neighboring actions in both
left-to-right and right-to-left layouts.

The volume gesture uses a 60dp edge zone along the album art. The
centre swipe resumes paused playback with an upward gesture and pauses
playing audio with a downward gesture.