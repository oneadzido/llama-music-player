# Llama Music Player

A privacy-respecting Android music player with a custom playback
engine, a sixty-preset equalizer, metadata editing, playback
scoping, and sixteen languages.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Android](https://img.shields.io/badge/Android-6.0%2B-brightgreen.svg)](https://developer.android.com)
[![Privacy](https://img.shields.io/badge/Privacy-Policy-blue.svg)](PRIVACY.md)

[![Download Latest Release APK](https://img.shields.io/badge/Download-Latest_Release_APK-brightgreen?style=for-the-badge&logo=android&logoColor=white)](https://github.com/oneadzido/llama-music-player/releases/latest/download/LlamaMusicPlayer-1.6.5.apk)

[![Build](https://github.com/oneadzido/llama-music-player/actions/workflows/main.yml/badge.svg)](https://github.com/oneadzido/llama-music-player/actions/workflows/main.yml)

---

## Latest Release: v1.6.5

Version 1.6.5 keeps the seek bar visible in a short undocked
player window. The song-info group and the seek-bar section
share the space between the top bar and the transport row
equally, instead of the seek-bar section being hidden there.
The timer row below the SeekBar shows only when the seek-bar
section can show it in full.

- **The song-info group and the seek-bar section share the
  short undocked window.** In `MusicPlayer.applyAlbumArtSize`,
  `shareSpace` is `albumArtHidden && !dockSongInfo`. That
  value is passed to `applySongInfoLayout(albumArtHidden,
  dockSongInfo, shareSpace)`, the one writer for this layout.
  When `shareSpace` is true, `mSongInfoGroup` and
  `mSeekBarSection` each take 0dp height with weight 1, with
  an 8dp gap between them — `SONG_INFO_SEEK_BAR_GAP_DP`
  as the seek-bar section's top margin — and each keeps its
  content centred in its half. The pair as a group fills the
  space between the top bar and the transport row.
- **The timer row shows only when it fits in full.** The
  timer row (`mTimeRow`, `R.id.timeRow`) holds the
  current time and the total duration below the SeekBar. In
  a full-size window it always shows: the seek-bar section
  hugs its content. In a short window where the section is
  given a fixed share, `MusicPlayer.updateTimeVisibility`
  compares the section height with what the row needs: the
  SeekBar height, the row's 8dp top margin, the row's height,
  and the section's top and bottom padding. When the section
  is shorter, the row is hidden (`GONE`) and the SeekBar
  stays, centred in its share. When space returns, the row
  returns. The check runs after `applySongInfoLayout` and
  again whenever the section's height changes, so dragging
  the split-screen divider re-evaluates it. There is no
  orientation rule and no hard-coded height threshold: the
  decision is a measurement of what fits.
- **Full-size layout is unchanged; the docked layout keeps
  its shape and follows the same timer rule.** In a
  full-size window the group takes the leftover space with
  weight 1 and the seek-bar section hugs its content below
  it. In a short, wide landscape window the group still docks
  beside the transport row through `setSongInfoDocked`, and
  the seek-bar section takes what is left above it. Because
  the docked seek-bar section is also given a fixed share,
  its timer row shows only when it fits in full, the same
  rule as in the shared layout.

## Preserved

- No change to the 1.6.4 resync: `mThemeRecreationPending`,
  `onLanguageChangedEvent`, `syncDisplayFromService`,
  `MusicService.requestPlayerResync()`,
  `MusicService.setActivityInForeground(boolean)`, and
  `UPDATE_PLAYER` are unchanged. `MusicService` stays at
  1.6.4.
- No change to the toast lane: 48dp
  (`TRANSPORT_ROW_SPLIT_BOTTOM_DP`) in split view and in
  landscape, plus the bottom buttons row's 16dp top padding.
- No change to the 1.6.4 button widths: top-bar buttons stay
  floored at 84dp, header and language buttons stay floored
  at 68dp, with no padding change.
- No change to ProGuard rules. The 1.6.5 change adds no new
  `BroadcastTranslator` subscriber and no new kept field. The
  timer check is a layout-change listener on the seek-bar
  section, and `mTimeRow` is a view reference.

## Previous Release: v1.6.4

Version 1.6.4 makes the player, equalizer, and editor update
instantly instead of waiting for a recreate or the next playback
state save, and makes long text truncate instead of scrolling
away. It fixes the case, seen on TECNO / HiOS, where equalizer
pitch and speed, the carousel, and the song info only refreshed
after an activity recreate. It also tightens the header and
language buttons from 76dp to 68dp.

- **Equalizer pitch and speed now reach the service whether or not
  it is playing.** `EqualizerActivity` dropped the `isPlaying()` gates
  (pitch, speed, and reset: five sites). A drag made while paused, in
  `READY`, or in `PREPARING` used to write the preference and never
  tell the service, so nothing changed until the next playback state
  save. The service already applies parameters in `READY` and before
  `start()`; now the equalizer actually sends them.
- **The player resyncs when it returns to the foreground.** The
  service re-broadcasts the full playback snapshot when the player
  goes from background to foreground, with `requestPlayerResync()` as
  a backup for a rebind where the flag did not flip, and the player
  pulls a full snapshot on bind and on resume. The carousel, song
  info, transport controls, and seek bar refresh without a recreate.
  `UPDATE_PLAYER` is posted, not sticky, so a player that was
  destroyed or backgrounded while a change broadcast fired used to
  miss it entirely.
- **Language recreate is deferred in `MusicPlayer`, like theme.**
  `onLanguageChangedEvent` parks the recreate in
  `mThemeRecreationPending` when the activity is not started, and
  `onStart()` runs it on return. Recreating a backgrounded activity
  can be deferred or dropped by aggressive OEMs, and the recreate can
  race the picker's teardown. Both actions are just a recreate, so
  they share one pending flag. Other activities still recreate
  immediately on a language change.
- **The Playlist search hint truncates with an ellipsis.** The hint
  only ellipsizes while horizontal scrolling is off, so the field now
  toggles: empty means hint showing, ellipsize on, scrolling off;
  typing means ellipsize off, scrolling on, so a long query scrolls
  instead of being cut off.
- **Metadata fields truncate in view mode and scroll in edit mode.**
  `setFieldsEnabled` now calls `applyFieldTruncation` for every field.
  In view mode the field truncates with "...": ellipsize on,
  horizontal scrolling off, the parent scroll wrapper disabled, and
  the field capped to the wrapper's measured width, which is what
  makes the ellipsis actually appear inside the wrapper's
  unspecified-width measure pass. In edit mode the field scrolls
  again. File info is never edited, so it is always truncated.
- **The seek bar in a short portrait window hides only when the layout
  genuinely cannot fit it.** A short, undocked window cannot fit
  header + group + seek bar + transport + bottom buttons, so the seek
  bar is the row that goes; the group stays, because it is the
  glanceable part. Everywhere else the seek bar stays visible. The
  group stays vertically centered in the space between the header and
  whatever follows it, with no extra top padding. The SeekBar keeps
  receiving progress updates while hidden, so returning to a
  full-size window shows the current position with no re-sync.
- **Header and language buttons move from 76dp to 68dp.** The Sort
  button in Playlist, the Scope button in Folder Manager, and the
  language button in the Credits dialog are 28dp high with 12sp text.
  Their minimum width drops from 76dp to 68dp, with no padding
  change. 76dp left too much empty width on buttons only 28dp high;
  68dp keeps short labels from collapsing without the extra width.

## Preserved

- No change to the 1.6.3 top-bar buttons: they stay floored at 84dp,
  with no padding change.
- No change to the 1.6.2 fixes: cold start still restores paused to
  READY, dialog window context and locale lifecycle still stay fixed,
  and equalizer and playback parameters still stay preserved across
  audio-session rebuilds.
- No change to scope matching, repeat, shuffle, sort, search filtering,
  favorite persistence, or click debounce.
- No change to fill-screen buttons: Preset and Reset in Equalizer and
  Import Folder in Folder Manager stay match_parent, and the Music
  Player bottom buttons stay 0dp with weight. Their width comes from
  layout, not from minWidth.
- No change to icon buttons: the Music Player transport buttons and the
  Playlist Clear button stay at their fixed sizes with minWidth 0dp.
- No change to button padding, height, text size, or click behavior.
- No change to ProGuard rules. The 1.6.4 changes add no new subscriber
  and no new kept field.

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
  lives in the Credits dialog.
- **Favorites:** Double-tap the album art, tap the heart in the
  song-info group on the player's screen, or tap a playlist row's heart.
- **Playback scope:** Restrict the player to favorites, loose tracks,
  or a chosen set of folders, genres, artists, or albums. The dialog
  shows a live count of the songs the selection would yield. The
  playing song stays in the queue regardless of scope.
- **Scope-aware playback navigation:** Manual Next and Previous,
  automatic song completion, and recovery from a prepare failure,
  player error, or playback stall all stay inside the active scope.
  A scope change mid-song takes effect when the current song ends.
  A scope with no songs pauses at the end of the current song
  rather than looping the previous one.
- **Resilient playback notification:** The playback notification
  survives a missing `POST_NOTIFICATIONS` grant on Android 13+, a
  sticky restart after a process kill, and a prepare failure's
  recovery. After a sticky restart its Play button works without
  opening the player, and it stays paused until you tap Play.
- **Resume after a phone call:** A call, a navigation prompt, or
  any other audio-focus interruption pauses playback, and playback
  resumes when focus returns. A tap, a next, a stop, or a volume
  change to zero during the interruption cancels the automatic
  resume.
- **A backgrounded volume key does not steal focus:** A hardware
  volume press while the player is backgrounded and another app is
  playing does not take audio focus back.
- **Audible resume from every surface:** A resume from the playback
  notification's play action, the lockscreen and media-session play
  callback, a headset media key, or an external intent is always
  audible, even when the music volume was at zero.
- **Foreground-aware player-screen resumption:** Volume-triggered
  and centre-swipe resumption from the player screen only work
  while the player screen is in the foreground. Music already
  playing in the background is unaffected.
- **Notification continuity through an interruption:** The
  playback notification, the shade entry, the lockscreen controls,
  and the media session survive a phone call and resume with the
  playback.
- **Automatic ducking:** A short interruption lowers the music's own
  output volume for its duration instead of pausing. The system
  stream volume is not touched.
- **Home button:** The main player screen's home button returns the
  app to the Android home screen on a deliberate two-tap gesture,
  leaving the music playing in the background. The first tap shows
  a toast; the second tap within one second moves the app to the
  background.
- **Centre swipe:** Swipe up on the centre of the album art to resume
  a paused song, or down to pause a playing one.
- **Album-art carousel:** Swipe left or right across the album art,
  or tap the transport buttons, to move between songs. Files opened
  from other apps transition through a cross-fade.
- **Volume gesture:** Swipe up or down along the album-art edge to
  change the music volume.
- **Volume-aware playback:** Music pauses at volume 0 and resumes
  when the volume is raised through the active player screen.
- **Live search:** A search bar at the top of the playlist filters
  on every keystroke, case-insensitively, across title, artist, album,
  and genre. The header shows the visible count over the total while a
  query is active. Clearing the query or pressing back
  restores the full list.
- **Playlist and scope fast-scroll:** Drag vertically along the
  leftmost or rightmost edge of a long list to scroll proportionally.
- **Sort progress feedback:** Cycling the sort mode shows the
  dim-and-spinner overlay until the reordered list has settled
  and drawn.
- **Library-load feedback:** A persistent overlay appears while the
  library loads on launch.
- **Metadata editor:** Edit ID3 tags, lyrics, and album art.
- **Folder management:** SAF folder import; removing a folder leaves
  every other folder's songs and every loose track alone. Each
  imported folder carries a badge identifying its storage medium.
- **10 sort modes:** Title A-Z, Title Z-A, Artist A-Z, Artist Z-A,
  Album A-Z, Album Z-A, Genre A-Z, Genre Z-A, Date (Newest First),
  Date (Oldest First).
- **32 theme colours.** Selecting one rebuilds every surface
  against the persisted colour. An activity that is backgrounded
  when the theme is picked still rebuilds when it returns.
- **Split-screen layout:** The album art resizes, shrinks, or hides
  with the window height; every screen adjusts its own content so
  the controls always fit.
- **Bluetooth support:** Headset controls, with pause on
  disconnect scoped to the case it describes. A disconnect pauses
  playback only when the headset was the active output and
  playback was running. A brief link flap that restores the route
  does not pause playback.
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
- **In-process events:** greenrobot, delivered by `BroadcastTranslator`

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
- FontAwesome by Fonticons, Inc.
- greenrobot by Markus Junginger and greenrobot (Apache License 2.0)

---

## License

Licensed under the MIT License.

Copyright (C) 2026 Richard Korbla Adzido.

---

Made with love in Ghana.