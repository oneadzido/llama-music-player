# Llama Music Player

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Android](https://img.shields.io/badge/Android-5.0%2B-brightgreen.svg)](https://developer.android.com)
[![Build](https://github.com/oneadzido/llama-music-player/actions/workflows/build-llama-apk.yml/badge.svg)](https://github.com/oneadzido/llama-music-player/actions/workflows/build-llama-apk.yml)

A high-quality Android music player with a custom-built playback engine featuring independent pitch and tempo control, a 60-preset equalizer, and comprehensive metadata editing capabilities.

---

## Table of Contents

1. [Features](#features)
2. [Technical Details](#technical-details)
3. [Audio Processing Architecture](#audio-processing-architecture)
4. [Download](#download)
5. [Building from Source](#building-from-source)
6. [Native Library Notes](#native-library-notes)
7. [Troubleshooting](#troubleshooting)
8. [License](#license)
9. [Acknowledgments](#acknowledgments)

---

## Features

### Audio Playback
- **Format Support**: MP3, FLAC, OGG, M4A, AAC, WAV
- **Pitch Control**: -6 to +6 semitones (independent of tempo)
- **Tempo Control**: 0.5x to 1.5x speed (independent of pitch)
- **High-Quality Processing**: WSOLA time-stretching algorithm via SoundTouch

### Audio Effects
- **10-Band Equalizer**: 60 presets including customizable bands
- **Bass Boost**: 0-100% adjustment
- **Surround Sound Effect**: 0-100% adjustment
- **Real-time Processing**: No audio gap when adjusting effects

### Library Management
- **Folder Import**: SAF (Storage Access Framework) folder selection
- **Metadata Editor**: Edit ID3 tags including title, artist, album, genre
- **Album Art Management**: Add, change, or remove album art
- **Automatic Scanning**: Recursive folder scanning for all audio files
- **Loose Tracks**: Support for individual audio files outside folders

### Playback Features
- **Sort Modes**: 10 sort modes including Title, Artist, Album, Genre, and Date
- **Repeat Modes**: Off, Repeat All, Repeat One
- **Shuffle Mode**: Random playback with history tracking
- **Background Playback**: Persistent notification with album art and controls
- **Lock Screen Controls**: MediaSession integration for lock screen playback control
- **Audio Focus Handling**: Proper ducking and interruption management
- **Bluetooth Support**: Headset controls with media button handling

### User Interface
- **Theme Colors**: 30 color options for UI customization
- **Album Art Display**: High-resolution artwork with rounded corners
- **Live Search**: Real-time filtering of songs
- **Credits Dialog**: Acknowledgment of open-source libraries
- **Toast Notifications**: User-friendly status messages

---

## Technical Details

**Minimum SDK**: Android 5.0 (API 21)

*Required for MediaSession lock screen controls, media style notifications, and proper audio focus handling. The SoundTouch native library version 2.1.0 itself would work on older Android versions (API 9+), but these modern features require API 21.*

**Target SDK**: Android 15 (API 35)

**Build Tools**: AGP 8.1.0, Gradle 8.0

**Java Version**: JDK 17 (build), Java 11 (compatibility)

**Media Framework**: Custom playback engine with SoundTouch

**Native Library**: SoundTouch 2.1.0 (official pre-built)

**Metadata Parsing**: jaudiotagger 3.0.1

**Icons**: FontAwesome 6 (Solid)

---

## Audio Processing Architecture

The app uses a dual-path audio processing architecture with intelligent fallback.

### Primary Path: SoundTouch Player (Default)

Audio File → AudioDecoder → SoundTouch → AudioTrack → Speakers

- **AudioDecoder**: Decodes MP3, AAC, FLAC, OGG to 16-bit PCM using MediaCodec
- **SoundTouch**: Official library for pitch and tempo shifting using WSOLA algorithm
- **AudioTrack**: Low-latency PCM audio output

### Fallback Path: MediaPlayer

Audio File → MediaPlayer → Speakers

- **Automatic Fallback**: If SoundTouch native library fails to load
- **Graceful Degradation**: User sees "MediaPlayer is active" toast
- **Persistence**: Fallback remains for the session to avoid retry loops

### Why Two Paths?

**SoundTouch Path** offers full pitch control (-6 to +6 semitones), full tempo control (0.5x to 1.5x), support for all audio formats, full equalizer attachment, low latency with AudioTrack priming, and works on API 21+.

**MediaPlayer Path** offers support for all audio formats, full equalizer attachment, but has limited pitch and tempo control that only works on API 23+. It also has higher latency compared to the SoundTouch path.

---

## Download

Get the latest APK from:

- **GitHub Releases**: Stable releases at https://github.com/oneadzido/llama-music-player/releases
- **GitHub Actions Artifacts**: Latest development builds from any successful workflow run

---

## Building from Source

### Prerequisites

- **Android Studio**: Hedgehog (2023.1.1) or later (recommended IDE)
- **JDK**: 17 (required for AGP 8.1.0)
- **Android SDK**: API 35 (platform tools)
- **Git**: Latest (for cloning)

> **Note**: NDK is **not required** because the app uses pre-built SoundTouch libraries.

### Local Build Instructions

```bash
# Clone the repository
git clone https://github.com/oneadzido/llama-music-player.git
cd llama-music-player

# Debug build (unsigned, for testing)
./gradlew assembleDebug

# Release build
./gradlew assembleRelease

# Clean build (removes all intermediates)
./gradlew clean