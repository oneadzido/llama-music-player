# Llama Music Player v1.4.5

**Release date:** November 11, 2026

Version 1.4.5 makes every resume audible, keeps progress messages
readable as they grow, and makes the notification show what the
player screen is showing. For the full change history, see the
`[1.4.5]` entry in the [CHANGELOG](CHANGELOG.md). For technical
specifications, installation, and usage, see the
[README](README.md).

---

## What's in this release

### Every resume is audible

Resuming a paused song from the player screen's centre-up swipe has
delivered sound even at music stream volume zero since v1.4.4. Every
other resume path — the playback notification's play button, the
lockscreen, a headset media key, an external app that asks the
player to resume — resumed the song silently when the music stream
was at zero. The player showed the play indicator; the user heard
nothing.

The audible-resume rule now lives in one place, in the player's
playback engine, and every resume path goes through it. On a device
whose music stream range is one or two steps, the service clamps to
the device maximum. From there, the volume gesture or a hardware key
takes over exactly as before.

### Toasts use the full display width

A toast was capped at a fixed width. Progress messages truncated
before the display was exhausted — an ellipsis in the middle of a
message whose full text would have fit on screen.

Toasts now span the display's usable width. A short message reads as
a short toast; a long message renders in full up to the usable span,
and ellipsizes only when it genuinely exceeds it. In a split-screen
window the toast is bounded by the window, not by the display.

### Progress messages stay readable as they grow

A persistent overlay — the message a long-running operation shows
while it runs — kept the width it had when it was first shown. A
message that started short and grew, like `Searching folders`
becoming `Searching folders · 123 songs found`, truncated at the
width from the moment it was created. The overlay now re-reads the
window's usable width before each update, so a growing message has
the room the window offers now.

The overlay composes its visible text once, inside
`ToastManager.composeOverlayText(title, body)`, from the two strings
the operation's owner supplies. The middle dot between the verb and
the count is the composition site's separator, not the caller's.

### The library notification carries a title and a count

The library notification used to be one message, split by the
notification into a title line and a body line. The split put the
separator in the title slot and left the count on the body line,
where a narrow notification shade truncated it before the display
was exhausted.

The notification now shows its verb phrase on the title line and its
count on the body line. The title reads `Searching folders`; the
body reads `123 songs found`, and the shade's expanded form shows
it in full. The in-app overlay shows the two joined once with a
middle dot, so the verb and the count read as two columns of one
status line. The composition site is `ToastManager.composeOverlayText`;
the notification's two slots are written by
`NotificationController.buildLibraryNotification(title, body)`.

### The playback notification and the player screen advance together

When the song changed, the player screen showed the incoming song
and the playback notification kept showing the outgoing one until
the next update — a favourite toggle, a repeat change, a playlist
refresh, a metadata edit, a completed scan. The two surfaces read
the same underlying song state through two different methods, and
the methods returned different songs for the same moment.

Both surfaces now read the same song from the same method. The two
advance together on every transition.