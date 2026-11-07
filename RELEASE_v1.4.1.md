# Llama Music Player v1.4.1

**Release date:** November 7, 2026

---

## Overview

Llama Music Player combines a scope-aware playback engine, persistent
media controls, and a compact interface designed to remain consistent
across all sixteen supported languages. Version 1.4.1 presents the
playback scope, notification lifecycle, and player controls as one
coherent experience.

## Playback and scope

The playback scope can select favorites, loose tracks, or specific
folders, genres, artists, and albums. Its live count uses the same
matching rules as the playback service. The current song and a pending
next song remain available while a scope change takes effect, allowing
the active track to finish naturally.

At song completion, the service chooses the next track from the active
scope. Repeat off advances through the scoped songs and pauses when
the scope is empty or the current song is its final entry. Repeat all
wraps within the scope, while shuffle selects from the same scoped
collection. Repeat one continues the current song.

## Playback notification

The playback notification is rendered from the service's committed
playback snapshot and remains synchronized with the media session.
Notification rendering and foreground-service promotion are handled
as distinct operations, so the notification can remain visible when
Android denies a foreground promotion because notification permission
is unavailable.

A sticky service restart rebuilds the notification from persisted
playback state. During a prepare or player error recovery, the service
keeps transition state visible to the notification controller until
reconciliation completes. Failed renders remain eligible for another
synchronization attempt, and the media session retains the same
committed snapshot as the notification.

## Player and dialog interface

The player uses a Home control for returning to the Android home
screen. The two-tap confirmation leaves playback running in the
background and uses a contrasting border and glyph color in both
light and dark themes.

The Home and Favorite controls use the standard 48dp control-button
treatment. The transport row follows a 48dp / 52dp / 56dp size
hierarchy with 28sp / 32sp / 36sp glyphs. The playlist favorite control
uses a 36dp surface and a 20sp glyph.

Dialog and activity title bars use logical start and end margins for
their 8dp title gutters. The same layout rule applies to the Playback
Scope, Language, Credits, Preset, and Theme dialogs and to the Player,
Playlist, Folder Manager, Metadata Editor, and Equalizer activities.
This keeps the title centered between its neighboring actions in both
left-to-right and right-to-left layouts.

The volume gesture uses a 60dp edge zone along the album art. The
centre swipe resumes paused playback with an upward gesture and pauses
playing audio with a downward gesture.

## Localization

The application includes English and fifteen additional locales:
Arabic, Chinese, Dutch, French, German, Hindi, Indonesian, Italian,
Japanese, Korean, Polish, Portuguese, Russian, Spanish, and Turkish.
The in-app language picker is available from the Credits dialog.
Changing the language rebuilds the visible interface without requiring
an application restart.

Every locale resource file follows the master resource arrangement.
Resource names and format arguments remain consistent across the
translation set, while plural categories follow the grammar of each
language.

## Technical profile

- **Minimum Android version:** Android 6.0 (API 23)
- **Target Android version:** Android 15 (API 35)
- **Language:** Java 17
- **Build system:** Gradle with Android Gradle Plugin 8.2.2
- **Audio formats:** MP3, FLAC, OGG, M4A, AAC, WAV
- **Equalizer:** 10 bands and 60 presets, including Custom
- **Pitch range:** −6.0 to +6.0 semitones
- **Playback speed:** 0.50x to 1.50x
- **Bass boost and surround:** 0–100%

## Download

Download the APK from the Assets section of the v1.4.1 release.

## Installation

1. Download the APK file.
2. Allow installation from the source used to download the APK.
3. Open the APK file and complete installation.

## First-time setup

1. Open the Folder Manager from the main screen.
2. Import or select the folder containing your music.
3. Apply the selection to let the library scan the available tracks.

## How to use

| Feature | How to access |
|---|---|
| Favorites | Double-tap the album art, tap the player's heart, or tap a playlist row's heart |
| Playback scope | Open Folder Manager and select Scope |
| Home | Tap the Home control twice within one second |
| Equalizer | Open Equalizer from the main screen |
| Metadata Editor | Open Metadata from the main screen |
| Playlist | Open Playlist from the main screen |
| Themes | Open LLAMA, then Themes |
| Language | Open LLAMA, then the language control in the version header |
| Volume | Swipe along either edge of the album art |
| Play or pause | Swipe up or down through the centre of the album art |
| Previous or next | Swipe left or right across the album art, or use the transport controls |
| Playlist fast-scroll | Drag along either vertical edge of the playlist |
| Sort mode | Select Sort in the playlist header |

## Credits

- jaudiotagger by jthink Ltd
- Font Awesome by Fonticons, Inc.

## License

Licensed under the MIT License.

Copyright (c) 2026 Richard Korbla Adzido.

Made with love in Ghana.
