# Llama Music Player v1.4.3

**Release date:** November 8, 2026

Version 1.4.3 makes playback resume reliably after a phone call, keeps
the playback notification a foreground notification through the
interruption, scopes the Bluetooth-disconnect pause to the case it
describes, and makes the Home button's return-home confirmation a
colour the user sees. For the full change history of this release, see
the `[1.4.3]` entry in the [CHANGELOG](CHANGELOG.md). For technical
specifications, installation, and usage, see the
[README](README.md).

---

## What's in this release

### Resume after a phone call

An interrupted playback — a phone call, a navigation prompt, a
headset disconnect from the audio-focus stack's point of view — is
recorded once, as an interruption. When the system returns audio
focus, the service retries focus acquisition for up to six seconds.
The retry covers the period during which the telephony stack holds
focus through its own teardown, and it is what makes the resume
reliable on devices where a single request inside the gain callback
is refused.

A gain that arrives while the telephony stack is still in a call
mode schedules the resume for the first interval at which the stack
has released focus, rather than burning an attempt on a request
that is almost certain to fail.

A user action during the interruption — a tap, a next, a stop, a
volume change to zero, a media-session command — clears the
interruption record. The automatic resume does not fire after the
user has chosen otherwise.

### Notification continuity through an interruption

The playback notification remains a foreground notification while
an interruption is in progress and playback is resumable. The shade
entry, the lockscreen controls, and the media session stay present
for the duration of the call and resume with the playback. Without
the promotion, an interrupted pause demotes the notification and
the shade entry is eligible for reclamation while the call is up.

### Bluetooth-disconnect pause scoped to the case it describes

A Bluetooth disconnect pauses playback only when the headset was
the active output, no other interruption is already handling the
pause, and playback was running. A disconnect during a call, a
link loss on a paired-but-inactive device, and a disconnect while
already paused are ignored and logged.

A link flap that restores the route within 400 ms suppresses the
pause. The disconnect broadcast schedules the pause after a
settling window, and if the Bluetooth route is restored within the
window, the pause is suppressed. A user-initiated transition
during the window cancels the scheduled pause.

The disconnect broadcast now carries the disconnecting profile and
the state value. The service logs both on each gate the request
passes through; neither value gates the pause.

### Home button confirmation is visible

The main player screen's Home control is a deliberate two-tap
gesture: the first tap arms the return and the second confirms it.
The button now carries three visible states through the sequence.
The first tap turns it the arming colour; the second tap turns it
the theme colour, and the return to the home screen fires after a
half-second dwell, so the confirmation is a colour the user sees
rather than an intent that fires before the screen updates. A tap
on any other button during either window cancels the return and
restores the Home control to its inactive colour, so the user's
last tap is always the one that decides what happens.