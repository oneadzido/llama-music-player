# Llama Music Player v1.5.1

**Release date:** November 17, 2026

Version 1.5.1 ships two correctness fixes to the surfaces a 1.5.0
user touches most: the transport controls now respond on the same
main-thread turn as the state change that updated them, and a
language change now rebuilds the app on the same path a theme
change does. The Credits dialog's open-and-close gesture keeps its
existing `slide_out_down.xml` and `slide_in_up.xml` animations, but
now drives them from one animation callback per half, so the two
halves of the gesture start on the same frame. The change is
internal; no user-facing behaviour the app had in 1.5.0 is removed.
For the full change history, see the `[1.5.1]` entry in the
[CHANGELOG](CHANGELOG.md). For technical specifications,
installation, and usage, see the [README](README.md).

---

## What's in this release

### The transport controls respond on the turn they are written

The play/pause and favorite buttons were written inside the
album-art carousel's commit runnable, which runs on the frame the
strip settles on. A transition that the user had just caused
therefore waited for the album art before it reached the buttons,
and on a slower input pipeline the wait was long enough to read
as the button not responding. The transport refresh now runs in
the subscriber's own main-thread turn, before the carousel is
touched. The album-art writes — the title, the artist, the seek
bar — stay in the carousel commit, where their timing is
correct.

The `updateUISafely` helper no longer posts a second runnable
when it is already on the main thread with no update in flight.
That is the common case: EventBus subscribers use the default
`POSTING` thread mode and run on the main looper, and click
listeners run on the main looper. The post path is kept for
updates that originate off the main thread.

### A language change rebuilds the app on the same path a theme change does

The language picker used to send a raw in-process broadcast
whose action string happened to match the translator's action
constant. The picker now posts a typed `LanguageChangedEvent`
through `BroadcastTranslator`, the same path every other stem
takes. The listener that applies the selection fires from the
picker's own `onDismiss` callback, which runs after the picker
window has detached, so the hosting activity's recreation cannot
race the picker's window teardown. The recreation runs through
`recreate()`, which re-runs `attachBaseContext` and therefore
re-applies the locale through `LanguageManager.wrap()`. No
manual `Resources.updateConfiguration` call precedes the
recreation; the outgoing activity's resources are not mutated on
their way out.

### The Credits dialog's open-and-close gesture starts on one frame

The gesture continues to use `slide_out_down.xml` and
`slide_in_up.xml`, the same XML animations every other screen
transition in the app uses. What changed is the wiring: the
outgoing slide runs on the player content and shows the dialog in
its `onAnimationEnd`; the incoming slide runs from the dialog's
`onDismiss`. Both halves are started by the framework's animation
callback rather than by two independent `postDelayed` calls, so
they cannot slip by a vsync relative to each other. The durations
continue to live in the XML resources; no Java-side duration is
kept in sync.