# Llama Music Player

A high quality android music player with a custom-built music playback engine with pitch/tempo control, 60-preset equalizer, and metadata editing.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Android](https://img.shields.io/badge/Android-4.4%2B-brightgreen.svg)](https://developer.android.com)
[![Build](https://github.com/oneadzido/llama-music-player/actions/workflows/build-llama-apk.yml/badge.svg)](https://github.com/oneadzido/llama-music-player/actions/workflows/build-llama-apk.yml)

## Features

- **Audio Formats**: MP3, FLAC, OGG, M4A, AAC, WAV
- **Pitch Control**: -6 to +6 semitones
- **Tempo Control**: 0.5x to 1.5x speed
- **10-Band Equalizer**: 60 presets including custom (adjustable)
- **Bass Boost & Surround**: 0-100% adjustment
- **Metadata Editor**: ID3 tags and album art
- **Folder Management**: SAF folder import
- **Search**: Live search with real-time filtering
- **Sort Modes**: Title, artist, album, genre, date (10 modes)
- **Theme Colors**: 30 color options
- **Bluetooth**: Headset support with media buttons
- **Background Playback**: Notification controls with album art

## Technical Details

- **Minimum SDK**: Android 4.4 (API 19)
- **Target SDK**: Android 15 (API 35)
- **Build Tools**: AGP 8.1.0, Gradle 8.0, JDK 17
- **Native Library**: SoundTouch 2.3.3 for pitch/tempo
- **Metadata**: jaudiotagger 3.0.1

## Credits

- **SoundTouch** library by Olli Parviainen
- **jaudiotagger** by jthink Ltd
- **FontAwesome** by Fonticons, Inc.

## Download

Get the APK from:
[https://github.com/oneadzido/llama-music-player/releases](https://github.com/oneadzido/llama-music-player/releases)

## System Requirements

- Android 4.4 (KitKat) / API 19 or higher
- 512 MB RAM minimum
- 50 MB storage

## Build from Source

### Prerequisites

- Android Studio Hedgehog (2023.1.1) or later
- JDK 17
- Android SDK 35
- NDK 25.1.8937393
- CMake 3.22.1

### Build Commands

```bash
git clone https://github.com/oneadzido/llama-music-player.git
cd llama-music-player

# Debug build
./gradlew assembleDebug

# Release build with optimizations
./gradlew assembleRelease