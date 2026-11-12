# Llama Music Player v1.4.6

**Release date:** November 12, 2026

Version 1.4.6 replaces in-place theme repaints with activity
recreation, so a theme change reaches every surface on every device.
It also closes a volume-key path that could take audio focus back
from an app that was playing. For the full change history, see the
`[1.4.6]` entry in the [CHANGELOG](CHANGELOG.md). For technical
specifications, installation, and usage, see the
[README](README.md).

---

## What's in this release

### Theme changes rebuild every surface

Selecting a new theme no longer repaints live views. The theme
manager persists the selection and broadcasts it; every activity
answers with a recreation; every open dialog answers by dismissing
itself. A theme change therefore produces the same guarantee on
every device: whatever surface was on screen is torn down and
rebuilt against the persisted color, and nothing that survived the
broadcast carries a stale drawable, a stale text color, or a cached
asset keyed by the previous color. The toast overlay rebuilds its
own text view through the same `createOverlayInView` path it uses on
first show. The notification is rebuilt from the persisted theme by
the recreated activity's next commit.

This mirrors the language picker's mechanism, which has been
reliable since 1.3.0. Both broadcasts now share one receiver body
per activity and one cancel-or-dismiss rule per dialog, so the two
paths cannot diverge on any device.

### Bluetooth-disconnect pause no longer interrupts a link flap

A Bluetooth disconnect pauses playback only when the headset was the
active output, no interruption was already handling the pause, and
playback was running. A restored route within the settling window
suppresses the pause. The pause is owned by the service, so it fires
even when the player activity is not alive.

### Audible resume from every surface

The audible-resume rule added in 1.4.4 lives in a single method in
the service. The player screen's centre-up swipe, the notification's
play action, the lockscreen and media-session play callback, a
headset media key, and an external intent all reach the rule through
the player's start, and all deliver sound at music stream volume
zero.

### A backgrounded volume key no longer steals focus

A hardware volume press while the player is backgrounded and another
app is playing can no longer take audio focus back. The volume-raise
path previously resumed whenever the service held focus **or** an
interruption record existed; the record exists precisely while the
interrupting app still holds focus, so the press could reach a resume
and request focus. The gate now requires the service to already hold
focus. The interrupting app keeps playing, and the `AUDIOFOCUS_GAIN`
that ends the interruption resumes the player through the ordinary
path.

The same rule covers the other volume-key paths the app has. A
Bluetooth headset remaps its volume keys to next and previous; that
remap refuses to act while focus is not held. A device that delivers
the volume key as a media-session play callback refuses the same way.

A successful resume now clears the interruption record, so a later
permanent loss from a different app is recorded rather than ignored.
Resume focus requests use `AUDIOFOCUS_GAIN_TRANSIENT`; the permanent
request is used only to acquire focus on first playback.