# Llama Music Player

A high-quality Android music player with a custom playback engine, pitch and playback speed control, a 60-preset equalizer, and metadata editing capabilities.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Android](https://img.shields.io/badge/Android-6.0%2B-brightgreen.svg)](https://developer.android.com)
[![Privacy](https://img.shields.io/badge/Privacy-Policy-blue.svg)](PRIVACY.md)

[![Download Latest Release APK](https://img.shields.io/badge/Download-Latest_Release_APK-brightgreen?style=for-the-badge&logo=android&logoColor=white)](https://github.com/oneadzido/llama-music-player/releases/latest/download/LlamaMusicPlayer-1.2.9.apk)

[![Build](https://github.com/oneadzido/llama-music-player/actions/workflows/main.yml/badge.svg)](https://github.com/oneadzido/llama-music-player/actions/workflows/main.yml)

---

## Origin Story

**Llama Music Player** is developed by a self-motivated, self-learning developer from Ghana.

He has always been passionate about computers since he was a child.

He develops Llama Music Player on his Android smartphone, in a deep partnership with AI.

He is not a developer by training. He is a developer by passion.

Read the full origin story here: [ORIGIN_STORY.md](ORIGIN_STORY.md)

---

## Download

[Download Latest Release APK](https://github.com/oneadzido/llama-music-player/releases/latest/download/LlamaMusicPlayer-1.2.9.apk)

Or browse the release page:

- [GitHub Releases](https://github.com/oneadzido/llama-music-player/releases/latest)
- GitHub Actions artifacts

---

## Features

- Audio Formats: MP3, FLAC, OGG, M4A, AAC, WAV
- Pitch Control: -6.0 to +6.0 semitones
- Playback Speed Control: 0.50x to 1.50x
- 10-Band Equalizer with 60 presets including Custom
- Bass Boost & Surround: 0-100% adjustment
- Album-Art Carousel: Swipe left or right across the album art to move to the previous or next song
- Button-Driven Slide: The transport buttons animate the same carousel with the same Material Design 3 curves the swipe uses, so the visual transition matches whether the user swiped or tapped. The service names the direction and the restart of every transition it begins, publishes them alongside the window on the same broadcast, and the carousel renders the animation they describe. The strip's first frame and the audio's transition are the same event
- Stable Neighbour Slots: The neighbour the carousel renders and the target the next commit advances to are the same value by construction. The two shuffle peek directions hold independent memos, so both the previous slot and the next slot show the art the corresponding button press will reach, whether the shuffle history is full, empty, or somewhere in between
- Incoming-File Reveal: Opening an audio file from another app transitions the album art through the same carousel; because the file can land anywhere in the rebuilt playlist, the strip runs a cross-fade — the outgoing art fades out on the emphasized accelerate curve, the incoming art fades in on the emphasized decelerate curve with a subtle scale-up. The cross-fade begins only once the incoming art has resolved, so the user sees one transition from the outgoing art to the incoming art and never a placeholder in between
- Volume Gesture: Swipe up or down along the album-art edge to change the music volume; the toast echoes at the device's volume limits
- Volume-aware Playback: Music pauses at volume 0 and resumes when the volume is raised; the gesture works from the moment a song loads, and the hardware volume key is live once the player has been opened
- Auto-Resume After Transient Focus Loss: A phone call, a navigation prompt, or a WhatsApp video status that takes audio focus for a bounded interval pauses the music; when focus returns, playback resumes from where it paused. A permanent loss is respected: playback stays paused
- Automatic Ducking: A short interruption that offers to share the output — a navigation prompt, a notification tone, a voice assistant reply — lowers the music's own output volume for its duration instead of pausing. The music keeps playing underneath and returns to full volume when the interruption ends; the system volume slider does not move
- Playlist Fast-Scroll: Drag vertically along the playlist edge to move through a long library in a bounded number of gestures
- Sort Progress Feedback: Cycling the sort mode shows the same dim-and-spinner overlay the app uses for longer operations, so a fast sort still reads as in-progress
- Metadata Editor: Edit ID3 tags and album art
- Folder Management: SAF folder import; removing a folder leaves every other folder's songs and every loose track alone
- Live Search: Real-time filtering
- 10 Sort Modes: Title, artist, album, genre, date
- 30 Theme Colors to customize the UI
- Split-screen Layout: The album art resizes, shrinks, or hides with the window height; when it hides, the title and artist group moves up to occupy the space the album art vacated, the artist line stays visible, and the transport row's bottom padding is reduced so the group's slot has room for both lines; the equalizer scrolls its band and effect controls together as one column; the other screens adjust their own content the same way, so the controls always fit
- Bluetooth Support: Headset controls
- Background Playback: Persistent notification with album art
- Lock Screen Controls: MediaSession integration

## Technical Details

- Minimum SDK: Android 6.0 (API 23)
- Target SDK: Android 15 (API 35)
- Build Tools: AGP 8.2.2, Gradle 8.2, JDK 17
- Metadata Parsing: jaudiotagger 3.0.1
- Icons: FontAwesome 6 (Free Solid)

## How It Works

<details>
<summary><b>Inside the codebase — seven subsystems worth knowing about</b></summary>

Seven subsystems are worth knowing about when reading the source.

**The playback state machine** lives in `MusicService`. A single
`ServiceState` enum, two monotonic counters, and a two-index model
(`mCurrentIndex` / `mTargetIndex`) hold every transition-related value.
State mutations run on the main thread; raw `MediaPlayer` calls run on
a dedicated `HandlerThread`. Only marshalled `Runnable`s cross between
them. The `READY` state is the state a song sits in when it has been
prepared and is waiting for the play button.

Six rules belong to this machine and are worth stating explicitly.
First, the music-stream volume receiver is a facet of it: registered in
`onBind(Intent)` and unregistered in `onDestroy()`, so once the user
has opened the player a drop to zero pauses and a raise above zero
resumes or starts. Second, a pause caused by a transient audio-focus
loss is remembered; when the framework returns `AUDIOFOCUS_GAIN` and
the machine is still paused, playback resumes from the paused
position. A permanent loss is not remembered: the user has moved on,
and playback does not restart on its own. Third, a duck request —
`AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK`, what a navigation prompt or a
notification tone sends — does not pause. The machine lowers the
MediaPlayer's own output volume to `DUCK_VOLUME_FRACTION` for the
duration of the interruption and restores it to full on
`AUDIOFOCUS_GAIN`. The system stream volume is not touched, so the
volume slider does not move and the receiver is not involved. Fourth,
volume-up at the device maximum does not emit `VOLUME_CHANGED_ACTION`
— no value changed — so `MusicPlayer` overrides `dispatchKeyEvent` to
resume a paused song from the foreground when the volume is already at
max. Fifth, the foreground path is the only one the activity can offer
for that case. The framework does not deliver volume keys to a
backgrounded activity, and no broadcast fires at the maximum because
no value changes. When the player is backgrounded and the song is
deliberately paused, resuming goes through the notification's play
action, which reaches the service through the media session in every
screen state. That is the platform's intended affordance for the case,
and it is what the app relies on. Sixth, every transition the machine
begins names its direction and its restart as properties of the
generation that produces it. A next or previous request notes `+1` or
`-1`; an automatic advance notes `+1`; a reprepare of the same song
notes `0`; a wrap that restarts the current song sets the restart
flag. The values are consumed into the generation when the transition
starts and published alongside the window on the same broadcast, so
the carousel's animation and the audio's transition are the same
value by construction. A headset disconnect is a seventh facet of the
same machine: the service registers a receiver for the pause request
in `onCreate` and unregisters it in `onDestroy`, so the pause fires
whether or not the player activity is alive.

**The playback window** is the single source of truth for the album
art. `PlaybackWindow` is an immutable value holding three paths — the
previous neighbour, the current song, the next neighbour — and the
service's monotonic transition counter. `MusicService.getWindowSnapshot()`
is the sole producer; it computes the window from two pure functions,
`peekNextIndex(int)` and `peekPreviousIndex(int)`, that are the same
primitives the service's own next and previous requests commit from.
Under shuffle, the two peek functions hold independent memos, so the
answer the window advertises in each direction survives a window read
and is the index the matching commit will play, in both the
history-backed and the fallback case. The window itself is memoised
for the current state, so two consecutive reads with the same state
return the same three paths: the neighbour slot the carousel renders
and the target the next commit advances to are the same value by
construction. The peek memos, the shuffle history, and the window
cache are index-keyed and are cleared on every operation that changes
what an index means. The carousel is the sole consumer; it holds no
playlist, no index, and no bitmap, and it adopts every window through
a single method. Alongside the window, the broadcast carries the
transition's direction (`+1`, `-1`, or `0`) and its restart flag. The
direction and the restart are properties of the generation the window
carries, so a receiver that has the window has the transition that
produced it.

**The overlay stack** lives in `OverlayManager` and `ToastManager`.
`OverlayManager` is the process-wide, reason-keyed dim-and-spinner
overlay. `ToastManager` is the operation-stack manager for persistent
overlays and temporary toasts. An operation is registered on the stack
by its ID; the stack renders the top operation and retains the lower
ones. Cancelling an operation is removing its entry from the stack,
which is indistinguishable from an operation that completed. A toast's
vertical position is chosen from the window's current height: the
manager holds two margins and picks the one that centres the toast in
the visible gap for the window the app is running in, so a toast is
centred in full-screen and in split-screen alike, in every activity,
without any activity having to announce its own layout state.

**The album-art carousel** lives in `AlbumArtCarousel`. It is a pure
renderer of the window the service publishes. A three-slot strip one
slot-width wide sits inside a viewport one slot wide; the strip's
middle slot is aligned to the viewport at rest. A horizontal drag
follows the finger, and on release the strip either commits to a
neighbour or settles back to rest. Every other transition — a
transport button, a headset key, an automatic advance, an incoming
file — is animated from the service's broadcast alone. The activity
forwards the window, the direction, and the restart the service named
to the carousel's `setWindow` method, and the carousel renders exactly
the animation they describe. A commit presets the confirmed window
before the slide starts, so the animation's end action adopts the
window synchronously and the strip never enters the snapshot-awaiting
state. A restart adopts the incoming window from the first frame and
runs the below-threshold nudge. Both legs of the nudge run on the
same spring constants — the outward leg carries the strip to the
displaced position and the return leg carries it back over the same
distance, on the same damping ratio and stiffness — so the two halves
of the gesture take the same time to settle and overshoot by the same
small amount. Every motion toward a rest position — the nudge's two
legs, a returned swipe, a refused press, a snapshot timeout — runs on
the same spring, so the strip's physical settle is the same wherever
the motion began. A non-neighbour change runs the cross-fade: the
outgoing art fades out on the emphasized accelerate curve and the
incoming art fades in on the emphasized decelerate curve with a
subtle scale-up, over the same total duration a commit takes. The
cross-fade waits for the incoming art to resolve before it begins, so
the outgoing art stays on screen while the incoming art is being
decoded and the transition the user sees is one cross-fade from the
outgoing art to the incoming art, never a placeholder in between.
Every transition is animated, and the animation is chosen from the
shape the service named: a change to a neighbour slides a full slot
width on the emphasized decelerate curve; a restart of the same song
with an advanced generation runs a below-threshold nudge driven by
spring physics with a small overshoot; a change to a path that is not
one of the previous window's neighbours runs the staggered
cross-fade, with an accelerate outgoing leg and a decelerate incoming
leg. Every animation ends in a single main-thread turn that renders
the carousel's new bitmaps and hands the same turn to the activity's
commit runnable, so the album art, the title, the artist, the seek
bar, the transport glyph, the notification, and the media session
land on one frame.

**The playlist fast-scroll** lives in
`PlaylistActivity.setupEdgeFastScroll()`. It installs an
`OnItemTouchListener` on the playlist `RecyclerView` that claims a
vertical drag once the finger has crossed the touch slop inside the
list's leftmost or rightmost 32dp. The drag then maps linearly onto
the total scrollable range: a full viewport-height of finger travel
moves through five percent of the range. The gesture engages only
when the content is at least three times taller than the viewport; on
a shorter list the edge zones defer to the RecyclerView's own
scrolling. A tap is never consumed.

**Split-screen behaviour is per-activity.** Every activity declares
the same `configChanges` set and adjusts its own content when the
window resizes, instead of being recreated. `MusicPlayer` sizes,
shrinks, or hides the album art; when the album art hides, the
wrapper around it is hidden with it, the title-and-artist group is
pinned to the top of its weight slot at the album-art top's Y offset
so the title lands where the album art used to begin, and the
transport row's bottom padding is reduced so the group's slot has
room for the artist line. Every view below the group keeps its
position relative to the window bottom because the group's weight
slot absorbs the difference. `EqualizerActivity` toggles the band
wrapper's layout weight so the band controls take the space the
effect controls leave in the full layout, and take their natural
height in split mode — the band controls and the effect controls
share one scroll container, so in split mode the two sections scroll
together. `FolderManager` keeps the list visible in every window
size and shows its placeholder only when the list is empty.
`MetadataEditor` and `PlaylistActivity` let their existing scroll
containers absorb the overflow.

**The storage observer** lives in `StorageObserver`. It registers a
single content observer, for
`MediaStore.Audio.Media.EXTERNAL_CONTENT_URI`. A change notification
starts a quiet period; if no further change arrives inside the period,
the observer asks `PlaylistManager` to run a scan. Audio additions,
removals, and metadata edits — including the app's own edits, which
are flagged as self changes — trigger the scan. Image, video, and
download writes do not, because the audio MediaStore URI is the only
one observed.

</details>

## Build from Source

### Prerequisites

- Android Studio Hedgehog or later
- JDK 17
- Android SDK API 35

### Build Commands

```bash
git clone https://github.com/oneadzido/llama-music-player.git
cd llama-music-player

./gradlew assembleDebug     # Debug build
./gradlew assembleRelease   # Release build
./gradlew clean             # Clean build
```

## First Time Setup

1. Tap the MANAGER button on the main screen
2. Tap IMPORT FOLDER to select your music folder
3. Tap UPDATE to save your selection
4. The app will scan your folder and build the playlist

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

## Privacy

Llama Music Player does not collect, transmit, or store any personal information. See [PRIVACY.md](PRIVACY.md) for the full policy.

## Contact

**Richard Korbla Adzido**
oneadzido@gmail.com