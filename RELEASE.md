# Llama Music Player v1.4.5

**Release date:** November 11, 2026

Version 1.4.5 makes every resume audible at music stream volume
zero and gives toasts the full width of the display before they
truncate. For the full change history of this release, see the
`[1.4.5]` entry in the [CHANGELOG](CHANGELOG.md). For technical
specifications, installation, and usage, see the
[README](README.md).

---

## What's in this release

### Audible resume from every surface

The centre-up album-art swipe has resumed a paused song audibly at
music stream volume zero since v1.4.4. Every other resume surface
— the playback notification's play action, the lockscreen and
media-session play callback, a headset media key, and an external
intent — reached `PLAYING` while the stream stayed at zero, so the
user saw a playing indicator and heard nothing. The gesture promised
playback and delivered sound. The other surfaces promised playback
and delivered silence.

The audible-resume rule now lives once, in the service, and every
resume path goes through it. The service reads the music stream's
current volume on the same main-thread turn that starts the player.
When the stream is exactly zero, it writes `Math.min(3, maxVolume)`
before the first audible sample is emitted. Three steps is the
smallest value reliably audible across the device range the app
supports, and small enough that a user who had deliberately lowered
the volume before pausing is not surprised by a loud resume. On
devices whose music stream range is one or two steps, the write
clamps to the device maximum. The user can raise or lower from
there with the edge drag or a hardware key.

The write fires `VOLUME_CHANGED_ACTION`, which reaches the
service's volume rule. By the time the broadcast is delivered, the
state machine is already at `PLAYING`, so the rule's raise branch
falls through without re-entering the resume path. The two paths
agree, and the resume delivers what the button promised.

The player screen's centre-up gesture delegates to the same rule.
Its inline volume write is removed; the gesture's contract is
"resume the song," and the audible outcome is the service's. A
future resume surface cannot miss the rule because there is only
one implementation of it, and only one place to add a new resume
path that honours it.

### Toasts use the full display width

A toast was capped at a fixed 320dp and ellipsized anything past
that cap. Progress messages like `Searching folders... 123 songs
found` truncated well before the display was exhausted, so the user
saw an ellipsis in the middle of a message whose full text would
have fit on screen. The cap was a fixed number, not the width the
display actually offered, and it produced an ellipsis that read as
a bug rather than as a deliberate truncation.

The toast is now bounded by the window width minus the app's
standard 16dp inset on each side, floored at the content's own
measured width. A short message renders as a short toast. A message
longer than the previous 320dp cap but shorter than the display's
usable width renders in full. A message that genuinely exceeds the
display's usable width ellipsizes at that edge — the honest signal
for "this does not fit," which is what ellipsis was always meant to
be.

The toast remains single-line by design. The available width is
read from the content view's measured width when it has been laid
out, so a toast in a split-screen window is bounded by the window
rather than by the display, and a long message in a narrow pane
ellipsizes at the pane's edge rather than overhanging it.