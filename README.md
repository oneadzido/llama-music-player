# Llama Music Player

A privacy-respecting Android music player with a custom playback
engine, a sixty-preset equalizer, metadata editing, playback
scoping, and sixteen languages.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Android](https://img.shields.io/badge/Android-6.0%2B-brightgreen.svg)](https://developer.android.com)
[![Privacy](https://img.shields.io/badge/Privacy-Policy-blue.svg)](PRIVACY.md)

[![Download Latest Release APK](https://img.shields.io/badge/Download-Latest_Release_APK-brightgreen?style=for-the-badge&logo=android&logoColor=white)](https://github.com/oneadzido/llama-music-player/releases/latest/download/LlamaMusicPlayer-1.5.3.apk)

[![Build](https://github.com/oneadzido/llama-music-player/actions/workflows/main.yml/badge.svg)](https://github.com/oneadzido/llama-music-player/actions/workflows/main.yml)

---

## Latest Release

### v1.5.3

Version 1.5.3 improves dialog reliability, activity recreation handling,
and music-player interface responsiveness. It fixes an issue where
the Credits dialog's theme and language buttons could become
unusable after opening and closing their pickers without making a
selection. The pickers now notify the host through a dedicated
dismiss callback, allowing the Credits dialog to clear its picker-open
flags independently of selection handling.

This release also improves coordination between dialog dismissal and
activity recreation during language and theme changes.
Playback and favorite controls now explicitly refresh their visual
states after toggle actions, addressing cases where the underlying
action succeeded but the corresponding icon remained visually
unchanged.

- Fix: theme and language buttons remain usable after closing pickers without selection.
- Fix: play/pause and favorite icons explicitly reflect toggle actions.
- Improved: activity recreation handling guarded against duplicate requests via window decor view.

---

## Origin Story

Llama Music Player was created by a self-taught, AI-assisted software
developer from Ghana.

Driven by a lifelong passion for computers and music, he began
building on an Android smartphone, without a laptop or formal
programming training. Working alongside AI, he turned a simple idea
into a feature-rich Android music player through experimentation,
hands-on testing, problem-solving, and continuous improvement.

His philosophy, The Llama Way, is simple: keep the experience
intuitive, avoid unnecessary complications, and respect the
listener's privacy. The full privacy policy is in
[PRIVACY.md](PRIVACY.md).

He built with the tools he had, learned along the way, and turned his
vision into reality.

Read the full origin story here: [ORIGIN_STORY.md](ORIGIN_STORY.md)

---

## Features

- **Audio formats:** MP3, FLAC, OGG, M4A, AAC, WAV readable;
  MP3, FLAC, OGG, M4A, MP4 writable
- **Pitch control:** −6.0 to +6.0 semitones
- **Playback speed control:** 0.50x to 1.50x
- **10-band equalizer** with 60 presets, including Custom
- **Bass boost and surround:** 0–100% adjustment
- **Sixteen languages:** English, Español, Français, Deutsch,
  Italiano, Português, Русский, 中文, 日本語, 한국어, हिन्दी,
  العربية, Türkçe, Bahasa Indonesia, Nederlands, Polski. The picker
  lives in the Credits dialog. Selecting a language posts a
  `LanguageChangedEvent` through `BroadcastTranslator`; every
  subscribed activity answers with `recreate()`, which re-runs
  `attachBaseContext` and rebuilds the UI in the new locale. The
  process is not restarted.
- **Favorites:** Double-tap the album art, tap the heart beside the
  transport controls, or tap a playlist row's heart. The player's
  heart is a 48dp control button. The playlist row uses a 36dp
  control button with a 20sp heart glyph, matching the folder
  manager's compact control treatment.
  Playlist heart taps are staged in memory and committed in one batch
  when the playlist screen pauses.
- **Playback scope:** Restrict the player to favorites, loose tracks,
  or a chosen set of folders, genres, artists, or albums. The dialog
  shows a live count of the songs the selection would yield, computed
  with the same matcher the service uses. A filter mode with no values
  checked is a scope that matches nothing: the count reads zero, the
  mode and its list stay in place, and the selection is not rewritten
  into All Songs. APPLY is enabled when the selection differs from the
  scope the dialog opened with, so an empty selection from a different
  starting scope can be committed and re-applying the current scope
  cannot. The playing song and any pending next stay in the queue
  regardless of scope. The dialog's title uses logical layout margins
  to guarantee exactly 8dp of space from the CANCEL and APPLY buttons
  in every locale.
- **Scope-aware song completion:** When the current song ends, the
  next song is chosen from the active scope, not from the
  grace-period-extended current playlist. A scope change mid-song
  advances into the new scope under repeat off and shuffle, and wraps
  to the new scope's first song under repeat all. A scope that
  contains no songs — All Songs on an empty library, Favorites with
  none marked, or a specific-artists scope with no artists checked —
  pauses at the end of the current song rather than looping the
  previous one.
- **Resilient playback notification:** The playback notification
  survives a missing `POST_NOTIFICATIONS` grant on Android 13+, a
  sticky restart after a process kill, and the brief `IDLE` window
  during a prepare failure's recovery.
- **Resume after a phone call:** A call, a navigation prompt, or
  any other audio-focus interruption is recorded once, and playback
  resumes when focus returns. Focus acquisition is retried for up
  to six seconds, so the resume lands on devices where the
  telephony stack releases focus late. A user action during the
  interruption — a tap, a next, a stop, a volume change to zero —
  clears the record and the automatic resume does not fire.
- **A backgrounded volume key does not steal focus:** A hardware
  volume press while the player is backgrounded and another app is
  playing does not take audio focus back. The volume-raise path
  requires the service to already hold focus; an interruption record
  without focus is a record of an app that still owns the stream, and
  the volume key does not act on it. The same rule covers the
  Bluetooth headset's volume remap and the media-session play
  callback.
- **Audible resume from every surface:** A resume from the player
  screen's centre-up swipe, the playback notification's play action,
  the lockscreen and media-session play callback, a headset media
  key, or an external intent delivers sound at music stream volume
  zero. The service raises the stream to three steps above zero on
  the same main-thread turn that starts the player. On devices whose
  music stream range is one or two steps, the write clamps to the
  device maximum.
- **Notification continuity through an interruption:** The
  playback notification stays a foreground notification while an
  interruption is in progress and playback is resumable, so the
  shade entry, the lockscreen controls, and the media session
  survive a phone call and resume with the playback.
- **Automatic ducking:** A short interruption lowers the music's own
  output volume for its duration instead of pausing. The system stream
  volume is not touched.
- **Home button:** The main player screen's home button uses a Home
  icon and returns the app to the Android home screen on a deliberate
  two-tap gesture, leaving the music playing in the background. The
  first tap turns the button the theme colour and shows a toast;
  the second tap within one second turns it the pending colour and
  the app moves to the background half a second later, so the
  confirmation is a colour the user sees. A tap on any other button
  during either window cancels the return and restores the Home
  control to its inactive colour.
- **Centre swipe:** Swipe up on the centre of the album art to resume
  a paused song, or down to pause a playing one. The gesture requires
  twice the touch slop of travel to distinguish it from digitizer
  wobble, and commits on release rather than on each move, so a single
  drag toggles at most once. A swipe up at music-stream volume zero
  resumes the song through `MusicService.resume()`; the service
  raises the stream to three steps above zero on the same main-thread
  turn, so the resume is audible rather than silent.
- **Album-art carousel:** Swipe left or right across the album art, or
  tap the transport buttons, to move between songs. Files opened from
  other apps transition through a cross-fade.
- **Volume gesture:** Swipe up or down along the album-art edge to
  change the music volume.
- **Volume-aware playback:** Music pauses at volume 0 and resumes
  when the volume is raised.
- **Live search:** A search bar at the top of the playlist filters
  on every keystroke, case-insensitively, across title, artist, album,
  and genre. The header shows the visible count over the total while a
  query is active, and the sort button is hidden so a re-sort cannot
  disturb the filtered order. Clearing the query or pressing back
  restores the full list.
- **Playlist and scope fast-scroll:** Drag vertically along the
  leftmost or rightmost edge of a long list to scroll proportionally.
  The playlist and the playback-scope dialog share the same
  implementation, so the edge width, engagement threshold, and scroll
  rate are identical on both surfaces.
- **Sort progress feedback:** Cycling the sort mode shows the
  dim-and-spinner overlay, hidden only once the reordered list has
  settled and drawn.
- **Library-load feedback:** A persistent overlay appears while the
  library loads on launch.
- **Full-width toasts:** Toast messages span the full display width
  minus the app's overlay gutter — the standard 16dp inset every
  floating surface uses — on each side. A message longer than the
  previous 320dp cap but shorter than the display's usable width
  renders in full; a message that genuinely exceeds that span
  ellipsizes at the edge. The text inside the toast sits with an 8dp
  horizontal padding, so a compact floating label reads as a label
  rather than as a padded content card.
- **Growing progress messages:** A persistent overlay re-reads its
  available width from the window on every render, so a progress
  message that starts short and grows is only ellipsized when its
  composed content actually exceeds the current window's usable
  width. The overlay composes its visible text once, inside
  `ToastManager.composeOverlayText(title, body)`, from the two
  strings the operation's owner supplies. The middle dot between the
  verb and the count is the composition site's separator, not the
  caller's.
- **Library-notification titles and bodies as two strings:** The
  library notification renders its verb phrase and its count into
  two separate slots. `ToastManager` composes the in-app overlay's
  visible text once from those two strings, joined with a middle
  dot; the notification mirror receives the two strings separately
  and renders each into its own line. The composition site is
  `ToastManager.composeOverlayText`; the notification's two slots
  are written by
  `NotificationController.buildLibraryNotification(title, body)`.
- **Metadata editor:** Edit ID3 tags, lyrics, and album art. A write
  lands on the file the user is editing, whatever the entry path, and
  the post-write re-read is verification only.
- **Folder management:** SAF folder import; removing a folder leaves
  every other folder's songs and every loose track alone. Each
  imported folder carries a badge identifying its storage medium.
- **10 sort modes:** Title A-Z, Title Z-A, Artist A-Z, Artist Z-A,
  Album A-Z, Album Z-A, Genre A-Z, Genre Z-A, Date (Newest First),
  Date (Oldest First).
- **32 theme colours.** Selecting one rebuilds every surface — every
  activity, every open dialog, and the toast overlay — against the
  persisted color, on every device. No view is repainted in place.
- **Split-screen layout:** The album art resizes, shrinks, or hides
  with the window height; every screen adjusts its own content so the
  controls always fit.
- **Bluetooth support:** Headset controls, with the pause on
  disconnect scoped to the case it describes. A disconnect pauses
  playback only when the headset was the active output, no other
  interruption is already handling the pause, and playback was
  running. A link flap that restores the route within 400 ms
  suppresses the pause, and a user action during the window cancels
  it.
- **Background playback:** Persistent notification with album art.
- **Lock-screen controls:** `MediaSession` integration.

## Technical Profile

- **Minimum Android version:** Android 6.0 (API 23)
- **Target Android version:** Android 15 (API 35)
- **Language:** Java 17
- **Build system:** Gradle with Android Gradle Plugin 8.2.2
- **Metadata parsing:** jaudiotagger 3.0.1
- **Readable audio formats:** MP3, FLAC, OGG, M4A, AAC, WAV
- **Writable audio formats:** MP3, FLAC, OGG, M4A, MP4
- **Icons:** FontAwesome 6 (Free Solid)
- **In-process events:** greenrobot EventBus 3.3.1, delivered by `BroadcastTranslator`

---

## Installation

1. Download the APK file.
2. Allow installs from this source when prompted.
3. Open the APK file and complete installation.

---

## First Time Setup

1. Allow access to your music files when prompted.
2. Tap the MANAGER button on the main screen.
3. Tap IMPORT FOLDER to select your music folders.
4. Tap UPDATE to let the app scan and build your playlist.

---

## Permissions

Android asks for sensitive permissions when the feature that needs
them is first used. The app declares the permissions it needs in the
manifest and requests the runtime grants at the appropriate point in
the experience.

- `READ_MEDIA_AUDIO` (Android 13+) / `READ_EXTERNAL_STORAGE`
  (Android 6.0–12) for music-library access.
- `POST_NOTIFICATIONS` (Android 13+) for playback and library
  notifications.
- `BLUETOOTH_CONNECT` (Android 12+) for headset connection state and
  media-button routing.
- `READ_MEDIA_IMAGES` (Android 13+) / `READ_EXTERNAL_STORAGE`
  (Android 6.0–12) when the Metadata Editor needs to read an image
  selected for album artwork.
- `FOREGROUND_SERVICE` and `FOREGROUND_SERVICE_MEDIA_PLAYBACK` are
  manifest permissions used to run the media-playback foreground
  service. The media-playback permission is a normal permission
  granted at install time; Android does not present it as a runtime
  permission.
- `WAKE_LOCK` and `MODIFY_AUDIO_SETTINGS` support uninterrupted
  playback and audio effects. Battery-optimisation exemption is
  offered through Android's system settings when the user chooses that
  option.

---

## How to Use

| Feature | How to Access |
|---------|---------------|
| Centre swipe to play or pause | Swipe up on the album art to resume, down to pause |
| Previous / next song | Swipe left or right across the album art, or tap the transport buttons |
| Volume gesture | Swipe up or down along the album-art edge |
| Favorites | Double-tap the album art, tap the heart on the player's screen, or tap a playlist row's heart |
| Folder Manager | Tap MANAGER button on main screen |
| Playlist | Tap PLAYLIST button on main screen |
| Playlist fast-scroll | Drag vertically along the left or right edge of the playlist |
| Sort mode | Tap SORT in the playlist header |
| Playback scope | Tap MANAGER, then SCOPE in the folder manager header |
| Equalizer | Tap EQUALIZER button on main screen |
| Metadata Editor | Tap METADATA button on main screen |
| Themes | Tap LLAMA button on main screen, then THEMES |
| Language | Tap LLAMA button on main screen, then the language button in the version header |
| Home | Tap the Home button on the main screen twice within one second |

---

## Credits

- jaudiotagger by jthink Ltd
- Font Awesome by Fonticons, Inc.

---

## License

Licensed under the MIT License.

Copyright (c) 2026 Richard Korbla Adzido.

---

Made with love in Ghana.