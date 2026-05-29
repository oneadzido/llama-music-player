# Llama Music Player

A high quality android music player with a custom-built music playback engine with pitch/tempo control, 60-preset equalizer, and metadata editing.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Android](https://img.shields.io/badge/Android-5.0%2B-brightgreen.svg)](https://developer.android.com)

[![Build](https://github.com/oneadzido/llama-music-player/actions/workflows/build-llama-apk.yml/badge.svg)](https://github.com/oneadzido/llama-music-player/actions/workflows/build-llama-apk.yml)

---

## Features

- **Audio Formats**: MP3, FLAC, OGG, M4A, AAC, WAV
- **Pitch Control**: -6 to +6 semitones (independent of tempo)
- **Tempo Control**: 0.5x to 1.5x speed (independent of pitch)
- **10-Band Equalizer**: 60 presets including custom (adjustable)
- **Bass Boost & Surround**: 0-100% adjustment
- **Metadata Editor**: Edit ID3 tags and album art
- **Folder Management**: SAF (Storage Access Framework) folder import
- **Search**: Live search with real-time filtering
- **Sort Modes**: Title, artist, album, genre, date (10 sort modes)
- **Theme Colors**: 30 color options for UI customization
- **Bluetooth**: Headset support with media button controls
- **Background Playback**: Persistent notification with album art and playback controls
- **Lock Screen Controls**: MediaSession integration for lock screen playback control

## Technical Details

- **Minimum SDK**: Android 5.0 (API 21)
- **Target SDK**: Android 15 (API 35)
- **Build Tools**: AGP 8.1.0, Gradle 8.0, JDK 17
- **Media Framework**: Media3 (ExoPlayer) 1.4.1
- **Native Library**: SoundTouch 2.3.3 for pitch/tempo
- **Metadata Parsing**: jaudiotagger 3.0.1
- **Icons**: FontAwesome

> **Note**: Minimum SDK changed from API 19 (Android 4.4) to API 21 (Android 5.0) due to Media3 requirements.

## Download

Get the latest APK from:
- [GitHub Releases](https://github.com/oneadzido/llama-music-player/releases)
- GitHub Actions artifacts (from any successful workflow run)

## Build from Source

### Prerequisites

- **Android Studio**: Hedgehog (2023.1.1) or later
- **JDK**: 17
- **Android SDK**: API 35
- **NDK**: 25.1.8937393
- **CMake**: 3.22.1

### Local Build Instructions

```bash
# Clone the repository
git clone https://github.com/oneadzido/llama-music-player.git
cd llama-music-player

# Debug build (unsigned, for testing)
./gradlew assembleDebug

# Release build (requires signing configuration)
./gradlew assembleRelease

# Clean build (removes all intermediates)
./gradlew clean