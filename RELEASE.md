# Llama Music Player v1.4.4

**Release date:** November 9, 2026

Version 1.4.4 makes the centre-up album-art swipe audible at music
stream volume zero. For the full change history of this release, see
the `[1.4.4]` entry in the [CHANGELOG](CHANGELOG.md). For technical
specifications, installation, and usage, see the
[README](README.md).

---

## What's in this release

### Audible centre-up swipe at volume zero

The centre-up swipe on the album art resumes a paused song. At music
stream volume zero, that resume used to reach `PLAYING` in the state
machine — the play glyph flipped, the notification advanced, the
media session reported playing — while the stream stayed at zero, so
the user saw a playing indicator and heard nothing. The gesture
promised playback and delivered silence.

The gesture now reads the music stream's current volume on release.
When the service is ready to resume and the stream is exactly zero,
the gesture writes `Math.min(3, maxVolume)` to the stream before it
dispatches the resume. Three steps is the smallest value that is
reliably audible across the device range the app supports, and small
enough that a user who had deliberately lowered the volume before
pausing is not surprised by a loud resume. On devices whose music
stream range is one or two steps, the write clamps to the device
maximum. The user can raise or lower from there with the edge drag or
a hardware key.

The write fires `VOLUME_CHANGED_ACTION`, which reaches
`MusicService.onMusicVolumeChanged` — the app's volume rule, where
zero is silence and non-zero is play. The explicit `resume()` that
follows reaches `PLAYING` first on the same main-thread turn; the
later broadcast finds the state already at `PLAYING` and falls
through without acting. The two paths agree, and the swipe delivers
what it promised.

The downward branch does not touch the volume. A downward swipe at a
non-zero volume pauses and keeps the volume where the user set it; a
downward swipe at zero finds the service already paused — the volume
rule paused it when the stream reached zero — so the gesture is a
no-op either way. There is no gap on the pause side to close.