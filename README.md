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
- Volume Gesture: Swipe up or down along the album-art edge to change the music volume
- Volume-aware Playback: Music pauses at volume 0 and resumes when the volume is raised
- Metadata Editor: Edit ID3 tags and album art
- Folder Management: SAF folder import
- Live Search: Real-time filtering
- 10 Sort Modes: Title, artist, album, genre, date
- 30 Theme Colors to customize the UI
- Split-screen Layout: Every screen scales its vertical spacing to the window height, and the album art resizes, shrinks, or hides so the controls always fit
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

Three subsystems are worth knowing about when reading the source.

**The playback state machine** lives in `MusicService`. A single
`ServiceState` enum, two monotonic counters, and a two-index model
(`mCurrentIndex` / `mTargetIndex`) hold every transition-related value.
State mutations run on the main thread; raw `MediaPlayer` calls run on
a dedicated `HandlerThread`. Only marshalled `Runnable`s cross between
them.

**The overlay stack** lives in `OverlayManager` and `ToastManager`.
`OverlayManager` is the process-wide, reason-keyed dim-and-spinner
overlay. `ToastManager` is the operation-stack manager for persistent
overlays and temporary toasts. An operation is registered on the stack
by its ID; the stack renders the top operation and retains the lower
ones. Cancelling an operation is removing its entry from the stack,
which is indistinguishable from an operation that completed.

**The album-art carousel** lives in `AlbumArtCarousel`. A three-slot
strip one slot-width wide sits inside a viewport one slot wide; the
strip's middle slot is aligned to the viewport at rest. A horizontal
drag follows the finger, and on release the strip either commits to a
neighbour or springs back. The carousel owns the strip, the three
`ImageView`s, the rolling window of decoded bitmaps, and the async
loading of the previous and next neighbours.

**The split-screen layout** lives in `SplitScreenHelper`. Each activity
constructs one instance, captures its hierarchy once, and reapplies
the scaling on every configuration change. The factor is a pure
function of `Configuration.screenHeightDp`. Four bands: 1.0, 0.75,
0.5, and 0.25.

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

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

## Privacy

Llama Music Player does not collect, transmit, or store any personal information. See [PRIVACY.md](PRIVACY.md) for the full policy.

## Contact

**Richard Korbla Adzido**
oneadzido@gmail.com