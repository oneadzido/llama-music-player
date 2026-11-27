# Llama Music Player

Llama Music Player is a privacy-respecting Android music player for managing and enjoying a local music collection.
It combines background playback, playlist and folder management, audio effects, metadata editing, and a customizable interface.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Android](https://img.shields.io/badge/Android-6.0%2B-brightgreen.svg)](https://developer.android.com)
[![Privacy](https://img.shields.io/badge/Privacy-Policy-blue.svg)](PRIVACY.md)

[![Download Latest Release APK](https://img.shields.io/badge/Download-Latest_Release_APK-brightgreen?style=for-the-badge&logo=android&logoColor=white)](https://github.com/oneadzido/llama-music-player/releases/latest/download/LlamaMusicPlayer-1.7.0.apk)

[![Build](https://github.com/oneadzido/llama-music-player/actions/workflows/main.yml/badge.svg)](https://github.com/oneadzido/llama-music-player/actions/workflows/main.yml)

---

## Overview

Llama Music Player is designed for listeners who want direct control over music stored on their device.
Playback runs through a dedicated service, while focused screens give you access to the library, playlists, folders, equalizer, and metadata tools.

## Features

### Playback and audio

- **Background playback:** Music keeps playing when the app is not in the foreground.
  You can control playback from the notification and the lock screen.
- **Multiple playback controls:** Control playback from the player, the notification, headset buttons, and supported media actions.
- **Album-art gestures:** Swipe across the artwork to change tracks, tap the centre to play or pause, and swipe along the edge to adjust volume.
- **Pitch and speed:** Adjust pitch from −6.0 to +6.0 semitones and playback speed from 0.50× to 1.50×.
- **10-band equalizer:** Shape audio with 60 presets, including a customizable preset, plus bass boost and surround controls.
- **Audio-focus handling:** Playback responds to calls and other audio interruptions by pausing, resuming, or ducking as appropriate.
- **Bluetooth controls:** Use headset media buttons.
  Playback pauses safely when a headset disconnects.

### Music library and playlists

- **Common audio formats:** Browse MP3, FLAC, OGG, M4A, AAC, and WAV files.
- **Playlist sorting:** Organize tracks using ten sort modes for title, artist, album, genre, and date.
- **Live search:** Filter playlists as you type across title, artist, album, and genre.
- **Favorites:** Mark tracks and open your favorites from the player or playlist.
- **Playback scopes:** Play the full library, favorites, loose tracks, or selections by folder, genre, artist, or album.
- **Folder import:** Add music folders using Android's Storage Access Framework.
- **Fast scrolling:** Move quickly through long playlist and scope lists.

### Metadata and presentation

- **Metadata editor:** Edit supported audio tags, lyrics, and embedded album artwork.
- **Sixteen languages:** English, Español, Français, Deutsch, Italiano, Português, Русский, 中文, 日本語, 한국어, हिन्दी, العربية, Türkçe, Bahasa Indonesia, Nederlands, and Polski.
- **Thirty-two theme colours:** Choose an interface colour to suit your preference.
- **Responsive layout:** Player screens adapt to portrait, landscape, and Android split-screen windows.

## Requirements

Llama Music Player runs on **Android 6.0 (API 23)** and later.
It targets **Android 15 (API 35)**.

## Installation

1. Open the [latest GitHub release](https://github.com/oneadzido/llama-music-player/releases/latest).
2. Download the APK attached to the release.
3. If prompted, allow installation from the app or browser you used to open the APK.
4. Open the APK and follow Android's installation instructions.

## First-time setup

1. Open Llama Music Player and grant music-library access when asked.
2. Tap **MANAGER** on the main player screen.
3. Tap **IMPORT FOLDER** and choose a folder that contains music.
   Repeat for any other folders.
4. Tap **UPDATE** to scan the folders you selected.
5. Return to the player, or open **PLAYLIST**, to browse and play your tracks.

## Using the app

- **Play or pause:** Swipe up or down in the centre of the album art, or tap the transport controls.
- **Change songs:** Swipe left or right across the album art, or tap Previous or Next.
- **Adjust volume:** Swipe up or down along the edge of the album art.
- **Manage favorites:** Double-tap the album art, or tap the heart control in the player or playlist.
- **Browse or search tracks:** Open **PLAYLIST** and type in the search field.
- **Sort tracks:** Open **PLAYLIST** and tap **SORT**.
- **Import music folders:** Open **MANAGER** and tap **IMPORT FOLDER**.
- **Choose a playback scope:** Open **MANAGER** and tap **SCOPE**.
- **Adjust audio effects:** Open **EQUALIZER**.
- **Edit tags or artwork:** Open **METADATA**.
- **Change theme or language:** Open **LLAMA** and tap the **Language** or **THEME** button.
- **Keep playback running:** On the main screen, tap the Home button twice within one second to send the app to the background.

## Permissions

Android asks for permissions when a feature needs them.
Depending on your Android version and the feature you are using, the app may ask for:

- **Music-library access:** `READ_MEDIA_AUDIO` on Android 13 and later, or `READ_EXTERNAL_STORAGE` on earlier supported versions.
- **Notifications:** `POST_NOTIFICATIONS` on Android 13 and later, for playback and library notifications.
- **Bluetooth connection access:** `BLUETOOTH_CONNECT` on Android 12 and later, for headset connection state and media-button handling.
- **Image access:** `READ_MEDIA_IMAGES` on Android 13 and later, or the applicable external-storage permission on earlier versions, when you select artwork in the metadata editor.

The manifest also declares service and audio-related permissions used for background playback and audio effects, including `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_MEDIA_PLAYBACK`, `WAKE_LOCK`, and `MODIFY_AUDIO_SETTINGS`.

## Project documentation

- [Release notes](RELEASE.md) — overview, fixes, and preserved behaviour for the current release.
- [Changelog](CHANGELOG.md) — release-by-release project history.
- [Architecture](ARCHITECTURE.md) — application components, responsibilities, communication, and playback design.
- [Privacy policy](PRIVACY.md) — data handling and permissions.
- [Origin story](ORIGIN_STORY.md) — the story behind the project.

## Credits

- [jaudiotagger](https://www.jthink.net/jaudiotagger/) by jthink Ltd.
- [Font Awesome](https://fontawesome.com/) by Fonticons, Inc.
- [greenrobot EventBus](https://github.com/greenrobot/EventBus) by Markus Junginger and greenrobot contributors.

Third-party license texts are included under `app/src/main/assets/licenses/`.

## License

Llama Music Player is licensed under the MIT License.
See the [LICENSE](../LICENSE) file for the full license text.

Copyright © 2026 Richard Korbla Adzido.