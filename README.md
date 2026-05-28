# Llama Music Player

A high quality android music player with a custom-built music playback engine with pitch/tempo control, 60-preset equalizer, and metadata editing.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Android](https://img.shields.io/badge/Android-4.4%2B-brightgreen.svg)](https://developer.android.com)

## Features

- **Audio Formats**: MP3, FLAC, OGG, M4A, AAC, WAV
- **Pitch Control**: -6 to +6 semitones
- **Tempo Control**: 0.5x to 1.5x speed
- **10-Band Equalizer**: 60 presets including **custom** (adjustable)
- **Bass Boost & Surround**: 0-100% adjustment
- **Metadata Editor**: ID3 tags and album art
- **Folder Management**: SAF folder import
- **Search**: Live search with 128 character limit
- **Sort Modes**: Title, artist, album, genre, date
- **Theme Colors**: 30 color options
- **Bluetooth**: Headset support
- **Background Playback**: Notification controls

## Credits

- SoundTouch library by Olli Parviainen
- jaudiotagger by jthink Ltd
- FontAwesome by Fonticons, Inc.

## Download

Get the APK from:
https://github.com/oneadzido/llama-music-player/releases/tag/v1.0.0

## System Requirements
- Android 4.4 (KitKat) / API 19 or higher
- 512 MB RAM minimum
- 50 MB storage

## Build from Source

**Prerequisites**: Android Studio, JDK 17, SDK 34, NDK 25.1.8937393, CMake 3.22.1

```bash
git clone https://github.com/oneadzido/llama-music-player.git
cd llama-music-player
./gradlew assembleRelease
