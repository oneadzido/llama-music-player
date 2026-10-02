# Llama Music Player

A high-quality Android music player with a custom playback engine, pitch and playback speed control, a 60-preset equalizer, and metadata editing capabilities.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Android](https://img.shields.io/badge/Android-6.0%2B-brightgreen.svg)](https://developer.android.com)
[![Privacy](https://img.shields.io/badge/Privacy-Policy-blue.svg)](PRIVACY.md)

[![Build](https://github.com/oneadzido/llama-music-player/actions/workflows/main.yml/badge.svg)](https://github.com/oneadzido/llama-music-player/actions/workflows/main.yml)

---

## Origin Story

**Llama Music Player** is developed by a self-motivated, self-learning developer from Ghana.

He has always been passionate about computers since he was a child.

He develops Llama Music Player on his Android smartphone, in a deep partnership with AI.

He is not a developer by training. He is a developer by passion.

Read the full origin story here: [ORIGIN_STORY.md](ORIGIN_STORY.md)

---

## Features

- Audio Formats: MP3, FLAC, OGG, M4A, AAC, WAV
- Pitch Control: -6.0 to +6.0 semitones
- Playback Speed Control: 0.50x to 1.50x
- 10-Band Equalizer with 60 presets including Custom
- Bass Boost & Surround: 0-100% adjustment
- Album-Art Carousel: Swipe left or right across the album art to move to the previous or next song
- Button-Driven Slide: The transport buttons animate the same carousel, so the visual transition matches the swipe
- Volume Gesture: Swipe up or down along the album-art edge to change the music volume; the toast echoes at the device's volume limits
- Volume-aware Playback: Music pauses at volume 0 and resumes when the volume is raised; the gesture works from the moment a song loads, not only after the first tap on play
- Playlist Fast-Scroll: Drag vertically along the playlist edge to move through a long library in a bounded number of gestures
- Sort Progress Feedback: Cycling the sort mode shows the same dim-and-spinner overlay the app uses for longer operations, so a fast sort still reads as in-progress
- Metadata Editor: Edit ID3 tags and album art
- Folder Management: SAF folder import
- Live Search: Real-time filtering
- 10 Sort Modes: Title, artist, album, genre, date
- 30 Theme Colors to customize the UI
- Split-screen Layout: The album art resizes, shrinks, or hides with the window height; the equalizer scrolls its band and effect controls together as one column; the other screens adjust their own content the same way, so the controls always fit
- Bluetooth Support: Headset controls
- Background Playback: Persistent notification with album art
- Lock Screen Controls: MediaSession integration

## Technical Details

- Minimum SDK: Android 6.0 (API 23)
- Target SDK: Android 15 (API 35)
- Build Tools: AGP 8.2.2, Gradle 8.2, JDK 17
- Metadata Parsing: jaudiotagger 3.0.1
- Icons: FontAwesome 6 (Free Solid)

## Architecture Notes

Five subsystems are worth knowing about when reading the source.

**The playback state machine** lives in `MusicService`. A single
`ServiceState` enum, two monotonic counters, and a two-index model
(`mCurrentIndex` / `mTargetIndex`) hold every transition-related value.
State mutations run on the main thread; raw `MediaPlayer` calls run on
a dedicated `HandlerThread`. Only marshalled `Runnable`s cross between
them.

**The playback window** is the single source of truth for the album
art. `PlaybackWindow` is an immutable value holding three paths — the
previous neighbour, the current song, the next neighbour.
`MusicService.getWindowSnapshot()` is the sole producer; it computes
the window from two pure functions, `peekNextIndex(int)` and
`peekPreviousIndex(int)`, that are the same primitives the service's
own next and previous requests commit from. Under shuffle, the peek
functions memoise their answer so the index the window advertised is
the index the service plays. The memo and the shuffle history are
index-keyed and are cleared on every operation that changes what an
index means. The carousel is the sole consumer; it holds no playlist,
no index, and no bitmap, and it adopts every window through a single
method.

**The overlay stack** lives in `OverlayManager` and `ToastManager`.
`OverlayManager` is the process-wide, reason-keyed dim-and-spinner
overlay. `ToastManager` is the operation-stack manager for persistent
overlays and temporary toasts. An operation is registered on the stack
by its ID; the stack renders the top operation and retains the lower
ones. Cancelling an operation is removing its entry from the stack,
which is indistinguishable from an operation that completed.

**The album-art carousel** lives in `AlbumArtCarousel`. It is a pure
renderer of the window the service publishes. A three-slot strip one
slot-width wide sits inside a viewport one slot wide; the strip's
middle slot is aligned to the viewport at rest. A horizontal drag
follows the finger, and on release the strip either commits to a
neighbour or springs back. The transport buttons drive the same
animation through `animateToNext()` and `animateToPrevious()`. A
commit is confirmed when the service's next window names the path the
strip slid toward, and refused when it does not; a refused commit
springs back.

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
shrinks, or hides the album art. `EqualizerActivity` toggles the
band wrapper's layout weight so the band controls take the space the
effect controls leave in the full layout, and take their natural
height in split mode — the band controls and the effect controls
share one scroll container, so in split mode the two sections scroll
together. `FolderManager` keeps the list visible in every window size
and shows its placeholder only when the list is empty. `MetadataEditor`
and `PlaylistActivity` let their existing scroll containers absorb
the overflow.

## Download

Get the latest APK from:

- [GitHub Releases](https://github.com/oneadzido/llama-music-player/releases)
- GitHub Actions artifacts

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