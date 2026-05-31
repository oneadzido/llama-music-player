# Llama Music Player

A high quality Android music player with custom playback engine, pitch/tempo control, 60-preset equalizer, and metadata editing.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Android](https://img.shields.io/badge/Android-5.0%2B-brightgreen.svg)](https://developer.android.com)

[![Build](https://github.com/oneadzido/llama-music-player/actions/workflows/build-llama-apk.yml/badge.svg)](https://github.com/oneadzido/llama-music-player/actions/workflows/build-llama-apk.yml)

---

## Features

- Audio Formats: MP3, FLAC, OGG, M4A, AAC, WAV
- Pitch Control: -6 to +6 semitones (independent of tempo)
- Tempo Control: 0.5x to 1.5x speed (independent of pitch)
- 10-Band Equalizer with 60 presets including custom
- Bass Boost & Surround: 0-100% adjustment
- Metadata Editor: Edit ID3 tags and album art
- Folder Management: SAF folder import
- Live Search: Real-time filtering
- 10 Sort Modes: Title, artist, album, genre, date
- 30 Theme Colors: Customizable UI
- Bluetooth Support: Headset controls
- Background Playback: Persistent notification with album art
- Lock Screen Controls: MediaSession integration

## Technical Details

- Minimum SDK: Android 5.0 (API 21)
- Target SDK: Android 15 (API 35)
- Build Tools: AGP 8.1.0, Gradle 8.0, JDK 17
- Native Library: SoundTouch 2.3.3
- Metadata Parsing: jaudiotagger 3.0.1
- Icons: FontAwesome 6 (Free Solid)

## Download

Get the latest APK from:
- [GitHub Releases](https://github.com/oneadzido/llama-music-player/releases)
- GitHub Actions artifacts

## Build from Source

### Prerequisites

- Android Studio Hedgehog or later
- JDK 17
- Android SDK API 35
- NDK 25.1.8937393
- CMake 3.22.1

### Build Commands

```bash
git clone https://github.com/oneadzido/llama-music-player.git
cd llama-music-player

./gradlew assembleDebug   # Debug build
./gradlew assembleRelease # Release build
./gradlew clean           # Clean build