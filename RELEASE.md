# Llama Music Player v1.5.8

**Release date:** November 21, 2026

Version 1.5.8 fixes the dialog-refresh failures: a theme change
picked while an activity is backgrounded, a preset whose band
controls never refreshed, and a notification that kept the old
theme and language until the next playback change. It also
aligns two dialog returns with the behaviour the theme and
language dialogs already had: reselecting the already applied
preset now returns without applying anything, and switching
scope modes no longer wipes the checked boxes of the mode left
behind. It also decides on screen rotation: the portrait lock is
lifted, so the app follows the device. Several locales
have labels longer than a portrait row comfortably fits.

Llama Music Player itself remains licensed under the MIT License.

For the full change history, see the `[1.5.8]` entry in the
[CHANGELOG](CHANGELOG.md). For technical specifications, installation,
and usage, see the [README](README.md).

---

## What's in this release

### Fixed: theme change no longer dropped for a backgrounded activity

Every activity's `THEME_COLOR_CHANGED` handler used to recreate
only when the activity was visible. A backgrounded activity hit
its visibility guard and returned without recording anything, so
the event was dropped and the activity kept the theme it was
constructed with until it was left and re-entered.

The handler now records the event in `mThemeRecreationPending`
instead of dropping it, and the recreate runs when the activity
returns: from `onResume()` for the resumed-gated activities,
from `onStart()` for `MusicPlayer`, whose guard is the started
gate. A theme change while the activity is visible still
recreates immediately, as before.

### Fixed: preset change refreshes the band controls

Picking a preset applied it through the manager but relied on
the listener notification to refresh the band SeekBars and the
preset button. When that notification was lost, the controls
kept showing the previous preset's bands even though the audio
had changed.

The preset callback now refreshes from the manager's current
state immediately after applying: the band levels are re-read
into the SeekBars and the preset button re-reads the preset
name. The dialog owns the selection; the host owns the refresh.

### Fixed: notification rebuilds in the new theme and language immediately

The service had a theme subscriber that did nothing and no
language subscriber at all, so the notification kept the
previous theme and locale until the next playback change
forced a commit.

The service now forces a `broadcastUpdate()` on both events,
which commits a fresh snapshot against the newly persisted
theme and locale on the same turn. The notification rebuilds
immediately. The controller's own theme handler stays a no-op
on purpose: committing from there as well would post the same
notification twice.

### Fixed: reselecting the current preset applies nothing

The preset listener applied every selection it received,
including a tap on the preset the dialog had opened with. The
theme and language hosts already guard that case: each
captures the index the dialog opened with and returns early on
a reselect, while the tap is still acknowledged by the picker's
close and the row's bounce.

The preset listener now has the same guard. Reselecting the
current preset closes the dialog and changes nothing.
Selecting a different preset applies as before.

### Fixed: scope mode switches preserve each kind's checked boxes

Switching scope modes used to clear every kind's checked
boxes. Checking artists, switching to genres, checking a
genre, switching back to artists: the artist checks were gone.

Each filter kind now keeps its own draft set while the dialog
is open, and returning to a kind restores its checks exactly
as they were left. Nothing commits on a mode switch — APPLY
commits only the selected kind's set — and Cancel / dismiss
throws the drafts away. Reopening starts from the applied
scope.

### Decision: screen rotation is allowed

The portrait value on all five activities — player,
playlist, folder manager, equalizer, metadata editor —
is replaced with `unspecified`, the platform value for
auto-rotation. The system decides the orientation,
honouring the user's auto-rotate setting. Rotating the
device rotates the app.

Nothing is lost on rotation: the shared configChanges set
already covers orientation, so the activity is not destroyed
and an open dialog is not dismissed. The layout re-applies
for the new window — the equalizer switches to its compact
split layout, the player shrinks the album art — and every
other view re-measures.

### Existing behavior preserved

No Java file changes were needed for rotation: the
groundwork was already in place. Locale handling is
untouched — the in-app language picker still recreates each
activity, and a system language change still recreates it.
The preset dialog still fires for every tap; the guard is in
the host listener, as it is for theme and language. The
intentional Reset button still returns pitch and speed to
defaults. Language changes still recreate each subscribed
activity immediately. No change to scope matching, repeat,
shuffle, sort, search filtering, favorite persistence, or click
debounce. No change to ProGuard rules: the new subscriber is
kept by the existing EventBus rule and the new fields by the
existing package wildcard.
