# Llama Music Player

A privacy-respecting Android music player with a custom playback
engine, a sixty-preset equalizer, metadata editing, playback
scoping, and sixteen languages.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Android](https://img.shields.io/badge/Android-6.0%2B-brightgreen.svg)](https://developer.android.com)
[![Privacy](https://img.shields.io/badge/Privacy-Policy-blue.svg)](PRIVACY.md)

[![Download Latest Release APK](https://img.shields.io/badge/Download-Latest_Release_APK-brightgreen?style=for-the-badge&logo=android&logoColor=white)](https://github.com/oneadzido/llama-music-player/releases/latest/download/LlamaMusicPlayer-1.5.0.apk)

[![Build](https://github.com/oneadzido/llama-music-player/actions/workflows/main.yml/badge.svg)](https://github.com/oneadzido/llama-music-player/actions/workflows/main.yml)

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
  lives in the Credits dialog and rebuilds the UI in the new locale
  without a restart.
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
- **Volume-aware playback:** Music pauses at volume 0 and resumes when
  the volume is raised.
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
- **Target Android version:** Android 16 (API 36)
- **Language:** Kotlin (JVM target 17)
- **Build system:** Gradle 8.9 with Android Gradle Plugin 8.7.2
- **Kotlin toolchain:** Kotlin 2.0.21, Parcelize 2.0.21
- **Compile / target SDK:** 36 / 36 · **Build-tools:** 36.0.0
- **Core library desugaring:** enabled (for `java.time` on API < 26)
- **Metadata parsing:** jaudiotagger 3.0.1
- **Cross-component signalling:** EventBus 3.3.1 with the
  library's reflective subscriber lookup
- **Readable audio formats:** MP3, FLAC, OGG, M4A, AAC, WAV
- **Writable audio formats:** MP3, FLAC, OGG, M4A, MP4
- **Icons:** FontAwesome 6 (Free Solid)

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

## How It Works

<details>
<summary><b>Inside the codebase — the subsystems and the cross-cutting
concern worth knowing about</b></summary>

The subsystems below are worth knowing about when reading the source,
and one cross-cutting concern touches every activity. The subsystems
match the list in `proguard-rules.pro`'s header, in the same order.

### The signalling contract

Every signal that used to travel between components as an Android
broadcast now travels on the process EventBus
(`com.ark.llama.event.LlamaEventBus`). The set of events is grouped by
producer surface:

- `MetadataEvent` — `Updated`, `GenreUpdated`.
- `PlayerEvent` — `Snapshot` (sticky), `ApplyScope`.
- `PlaylistEvent` — `LibraryReset` (sticky), `Refresh`, `Display`,
  `SyncStarted`, `SyncComplete`, `TriggerGenreLoading`, `SongAdded`.
- `SessionEvent` — `AudioSessionChanged`, `PlaybackParamsRefresh`.
- `SettingsEvent` — `LanguageChanged` (sticky), `ThemeColorChanged`
  (sticky).
- `TransportEvent` — 17 events covering action-shaped commands and
  data-shaped payloads.

`LlamaEventBus.init()` is called once from `LlamaMusicPlayer.onCreate`
before any manager whose constructor registers a subscriber. The bus
is built with the library's reflective subscriber lookup. The one
`@PublishedApi internal` constant is `TAG`, referenced by the two
public inline functions `latest<T>()` and `removeLatest<T>()`;
Kotlin rejects a public inline function that reads a non-public
member, and `@PublishedApi` opens exactly that one reference.

The system `VOLUME_CHANGED_ACTION` broadcast is the one exception. It
is the framework's signal for a hardware volume-key press;
`MusicService` is a consumer of the system, not a producer of an
internal event. The volume receiver stays registered in
`MusicService.onBind` and unregistered in `onDestroy`.

### 1. The playback state machine (`MusicService`)

A single `ServiceState` enum, two monotonic counters
(`mGeneration` and `mStartToken`), and a two-index model
(`mCurrentIndex` / `mTargetIndex`) hold every transition-related
value. State mutations run on the main thread; raw `MediaPlayer`
calls run on a dedicated `HandlerThread`. Only marshalled `Runnable`s
cross between them. The `READY` state is the state a song sits in
when it has been prepared and is waiting for the play button.

Nine rules belong to this machine.

First, the music-stream volume receiver is a facet of it: registered
in `onBind(Intent)` and unregistered in `onDestroy()`, so once the
user has opened the player a drop to zero pauses and a raise above
zero resumes or starts. A service created without a bind — a sticky
restart, a Bluetooth connect, a media-button press — never registers
the receiver, so a hardware volume press on those paths is not
treated as a transport control.

Second, the volume-raise path requires the service to already hold
audio focus. An interruption record without focus is a record of an
app that still owns the stream; the volume key does not act on it.
The same rule covers the Bluetooth headset's volume remap in
`handleMediaButton` and the media-session `onPlay` callback, so no
volume-key path can take focus back from an app that is playing.

Third, an audio-focus interruption is recorded once, as an
`Interruption`, on the first loss that interrupts a live transition.
A subsequent loss of a different kind during the same interruption
does not overwrite the record, so a dialer that sends
`AUDIOFOCUS_LOSS` for the ringtone and `AUDIOFOCUS_LOSS_TRANSIENT`
for the call produces one record, and the gain that follows resumes
from it. A later `AUDIOFOCUS_LOSS` does replace a stale
`TRANSIENT_LOSS` record, so a genuinely new permanent takeover is
recorded. The record is cleared by any user-initiated transition, by
a successful resume, and by a transition to idle. The record drives
three decisions: whether a gain resumes playback, whether the
playback notification stays a foreground notification through the
interruption, and whether a subsequent loss of a different kind is
ignored.

Fourth, the gain that ends an interruption goes through
`resumeAfterInterruption()`, which retries focus acquisition on a
bounded cadence — twelve attempts at five hundred milliseconds —
because the telephony stack holds focus through its own teardown
after a call ends, and a single request inside the gain callback
fails on a meaningful fraction of devices. A gain that arrives while
`AudioManager.getMode()` reports a call mode skips the first focus
attempt and enters the retry path directly. When the retry budget
is exhausted, the interruption record is left in place: the
notification stays foreground, the shade entry and lockscreen
controls survive, and the play button resumes through the ordinary
transport path. `doResume()` and `resumeAfterInterruption()` request
`AUDIOFOCUS_GAIN_TRANSIENT`; the permanent request is used only to
acquire focus on first playback.

Fifth, a duck request —
`AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK`, what a navigation prompt or a
notification tone sends — does not pause. The machine lowers the
`MediaPlayer`'s own output volume to `DUCK_VOLUME_FRACTION` for the
duration of the interruption and restores it to full on
`AUDIOFOCUS_GAIN`. The system stream volume is not touched, so the
volume slider does not move and the receiver is not involved.

Sixth, volume-up at the device maximum does not emit
`VOLUME_CHANGED_ACTION` — no value changed — so `MusicPlayer`
overrides `dispatchKeyEvent` to resume a paused song from the
foreground when the volume is already at max. When the player is
backgrounded and the song is deliberately paused, resuming goes
through the notification's play action, which reaches the service
through the media session in every screen state.

Seventh, every transition the machine begins names its direction and
its restart as properties of the generation that produces it. A next
or previous request notes `+1` or `−1`; an automatic advance notes
`+1`; a reprepare of the same song notes `0`; a wrap that restarts
the current song sets the restart flag. The values are consumed into
the generation when the transition starts and published alongside
the window on the same `PlayerEvent.Snapshot` event, so the
carousel's animation and the audio's transition are the same value
by construction.

Eighth, a Bluetooth disconnect is a facet of the same machine.
`BluetoothReceiver` posts `TransportEvent.Pause` on the bus when the
headset profile reports a disconnect; `MusicService.onTransportPause`
is the sole consumer. The pause is gated on three conditions, all
required: the Bluetooth device is the current audio route, no
interruption is already handling the pause, and the state is playing.
A disconnect during a call, a link loss on a paired-but-inactive
device, and a disconnect while already paused are ignored and logged.
When all three pass, the pause is confirmed after a
four-hundred-millisecond settling window, which distinguishes a
disconnect from a link flap; a restored route within the window
suppresses the pause, and a user-initiated transition during the
window cancels it.

Ninth, the audible-resume rule is a facet of the same machine. The
service reads the music stream's current volume on the same
main-thread turn that starts the player. When the stream is exactly
zero, it writes `Math.min(3, maxVolume)` before the first audible
sample is emitted. The rule is invoked at both and only both sites
in `MusicService` where `MediaPlayer.start()` is dispatched — the
auto-start branch of `onPrepared` and the start continuation of
`doResumeInternal` — so every resume surface (the player screen's
centre-up gesture, the notification's play action, the lockscreen
and media-session play callback, a headset key, an external intent)
goes through it. The write fires `VOLUME_CHANGED_ACTION`, which
reaches `onMusicVolumeChanged` after the state machine has already
reached `PLAYING`; the volume rule's raise branch falls through and
does not re-enter the resume path.

The same machine also owns the three-tier playlist and the incoming-
file hand-off.

The display-song rule lives on the same machine, in one method:
`getDisplaySong()` returns the target during `PREPARING` and the
current-index song in every other state. The carousel, the player
UI, the art prefetch, the notification, and the media session all
read that one method, so every surface begins advancing toward the
target on the same main-thread turn a transition begins and every
surface lands on the incoming song on the same frame the transition
commits. The commit path is `MusicService.broadcastUpdate`, which
posts a sticky `PlayerEvent.Snapshot` and calls
`NotificationController.commitFromBroadcast` inline with the same
snapshot. The notification, the media session, the player UI, and
the carousel advance together.

The service's collection declarations use Kotlin's `Set` and `Map`,
not `java.util.Set` and `java.util.Map`. `mLooseTrackPaths` is a
Kotlin `Set<String>`; `commitFavorites(changes: Map<String, Boolean>?)`
accepts a Kotlin `Map` and iterates it with
`for ((path, favorite) in changes)`. The `LruCache` for the
notification art is built with an
`object : LruCache<String, Bitmap>(size)` expression rather than a
trailing lambda, because the trailing lambda form is not valid for
the constructor overload the library exposes. The session-token
provider is installed with `{ getSessionToken() }` — a call to the
public accessor — not with `{ sessionToken }`, which would be a
property reference the class does not expose.

### 2. The three-tier playlist (`MusicService`)

`mFullPlaylist` is the immutable database reflection.
`mScopedPlaylist` is `mFullPlaylist` filtered by the active
`PlaybackScope`. `mCurrentPlaylist` is the active queue, which is
`mScopedPlaylist` plus the songs at `mCurrentIndex` and
`mTargetIndex` — the grace period that keeps a currently-playing
song and its pending next alive regardless of scope.

`rebuildPlaylists()` is the single mutator for the latter two. It
calls `refreshLooseTrackPaths()` first, so the loose-tracks scope
can recognise a loose song without a database round-trip per match.

`commitFavorites(Map)` writes the per-song favourite bit through
`SongDatabase.setFavorite(String, boolean)`, then rebuilds the
playlists. The favourite column is written only by that targeted
path and by the fresh-row branch of the insert; every other metadata
write goes through `SongDatabase.updateSongMetadata(Song)`, which
never touches the column, so a scan or a genre load cannot clobber a
user's favourites.

`onCompletion()` asks the scoped playlist, not the grace-period-
extended current playlist, whether there is a next song. A grace-
period survivor is a song the user did not ask for; its position in
the current playlist must not decide whether there is a next song.
Repeat off advances into the scope when the current song is not the
last scope song; repeat all wraps to the next scope song; shuffle
picks from the scope through `pickShuffleNextInScope(int)`; every
non-repeat-one mode pauses through `pauseAtEndOfQueue(int)` when the
scope is empty. Repeat one restarts the current song and never asks
the question.

### 3. The incoming-file hand-off (`MusicPlayer` → `MusicService`)

A user who opens an audio file from another app hands the path to the
activity, and the activity hands it to the bound service through a
direct method call —
`MusicService.requestIncomingFile(String)`. No broadcast, no
EventBus delivery window between the activity and the service for
the hand-off itself; the play result afterwards travels on the bus
through the ordinary `PlayerEvent.Snapshot`.

The service stores the path in `mPendingSongPath` and re-enters
reconciliation. A path that names a song already in the playlist
transitions to it immediately. A path that names a song not yet in
the playlist stays parked until the insert's rebuild adds it, and the
reconcile that follows the rebuild resolves it. The slot is cleared
only when the transition it names begins, so a request that arrives
while another transition is preparing survives the prepare and plays
after it settles. A path the insert rejected is cleared by the
insert's failure callback through
`MusicService.clearPendingSongPath()` before the next reconcile runs.

### 4. The album-art gesture classifier (`MusicPlayer`)

A single touch on the album-art container is classified on its first
significant move into one of five states:

- `TOUCHED` — no significant move yet.
- `CAROUSEL` — horizontal swipe.
- `VOLUME` — vertical swipe that started in the left or right edge
  zone.
- `PLAYBACK` — vertical swipe that started at the centre.
- `CANCELLED` — edge-zone swipe on a device with no music stream.

The classifier owns all four gestures the album art supports — the
carousel swipe, the volume drag, the centre swipe to play or pause,
and the double-tap favourite — and the four share one state enum so a
touch cannot fall into two modes at once.

The `PLAYBACK` gesture requires twice the touch slop of travel to
distinguish an intentional swipe from digitizer wobble on a resting
finger, and it commits on release, not on the first significant move,
so a single drag toggles at most once. A centre-up swipe resumes a
paused or ready song through `MusicService.resume()`. Whether the
resume is audible is the service's decision: the service raises the
music stream to an audible level when it is at zero, on the same
main-thread turn that starts the player. The gesture does not read or
write the stream volume. The downward branch pauses a playing song
and does not touch the volume.

The `TOUCHED` state is the resting state from which the double-tap
favourite is detected: a touch that never left the touch-slop radius
is a tap, and two taps within
`ViewConfiguration.getDoubleTapTimeout()` toggle the current song's
favourite through `handleGestureDoubleTap()`.

The transport buttons do not drive the carousel directly. They post
`TransportEvent.Next` or `TransportEvent.Previous` on the bus; the
service notes the direction and restart, sets the target, and
publishes a fresh `PlayerEvent.Snapshot`. The carousel animates from
that snapshot, in the same turn the window is committed.

The activity's listeners that are wired into the album-art carousel
use `object : AlbumArtCarousel.SwipeCommitListener` rather than a
SAM lambda, because `SwipeCommitListener` is a regular Kotlin
interface, not a `fun interface`. Kotlin's SAM conversion only
applies to Java interfaces and to `fun interface` declarations;
regular Kotlin interfaces require the explicit `object :`
expression. The same rule applies to
`PlaylistAdapter.FavoriteListener`,
`PlaylistAdapter.OnSongClickListener`,
`PlaybackScopeDialog.OnScopeAppliedListener`, and every other
listener declared in the app's own Kotlin source.

### 5. The playback window (`PlaybackWindow`)

`PlaybackWindow` is an immutable value holding three paths — the
previous neighbour, the current song, the next neighbour — and the
service's monotonic transition counter.

`MusicService.getWindowSnapshot()` is the sole producer. It computes
the window from two pure functions, `peekNextIndex(int)` and
`peekPreviousIndex(int)`, that are the same primitives the service's
own next and previous requests commit from. Under shuffle, the two
peek functions hold independent memos, so the answer the window
advertises in each direction survives a window read and is the index
the matching commit will play, in both the history-backed and the
fallback case.

The window itself is memoised for the current state, so two
consecutive reads with the same state return the same three paths:
the neighbour slot the carousel renders and the target the next
commit advances to are the same value by construction. The peek
memos, the shuffle history, and the window cache are index-keyed and
are cleared by `clearShufflePeek()` on every operation that changes
what an index means.

`AlbumArtCarousel` is the sole consumer; it holds no playlist, no
index, and no bitmap, and it adopts every window through
`setWindow(PlaybackWindow, int, boolean)`. Alongside the window,
the `PlayerEvent.Snapshot` event carries the transition's direction
(`+1`, `−1`, or `0`) and its restart flag. The direction and the
restart are properties of the generation the window carries, so a
receiver that has the window has the transition that produced it.

### 6. The playback scope (`PlaybackScope` and `PlaybackScopeDialog`)

`PlaybackScope` is an immutable value object with seven kinds — All
Songs, Favorites, Loose Tracks, Specific Folders, Specific Genres,
Specific Artists, Specific Albums — serialized to preferences as JSON
through `serialize()` and `deserialize()`, under the preference key
`playback_scope_json`. A malformed string deserialises to
`PlaybackScope.ALL_SONGS`, so a corrupted preference cannot brick the
player. A scope of one of the four filter kinds may carry an empty
set; that is a scope that matches nothing, serialized and
deserialized unchanged. Its four filter fields are Kotlin
`Set<String>` built with `.toSet()`, so the field type and the
serialized JSON agree on which interface is being carried across the
serialization boundary.

`PlaybackScopeDialog` presents the seven modes as a single flat list,
expands a mode's filter list below its row when the mode is selected,
and computes the song count the selection would yield from the same
matcher the service uses — `PlaybackScope.matches(Song, Set)` — over
the dialog's own cached song list and loose-track path set, loaded on
its `mLoader` executor in `loadDynamicFilterLists()`.

A filter mode with no values checked is a scope that matches nothing,
and the dialog presents it as such: the mode stays selected, its list
stays expanded, and the count reads zero. The selection is not
rewritten into All Songs; the All Songs scope is reached through the
All Songs row. APPLY is enabled when `buildScope()` differs from the
scope the dialog opened with — the kind and all four filter sets
compared — so an empty filter selection from a different starting
scope is a change the user can commit, and re-applying the current
scope is not a change. A preference change made during the commit
window cancels the queued commit and restores APPLY through the same
comparison.

The dialog's top bar carries CANCEL at the start, a title in the
middle with weight 1, and APPLY at the end. The title truncates with
an ellipsis and carries 8dp start/end padding, so a translation
longer than the available width truncates rather than running under
the CANCEL or APPLY label. The CANCEL and APPLY buttons are each
guarded by a 500 ms debounce; the APPLY button is additionally
guarded by `mApplyInProgress`, which refuses a second tap for the
whole window between the tap and the dismissal and is cleared if the
user changes a preference before the commit fires. A change made
during that window cancels the queued commit and restores the button
to the state the current selection warrants.

The dialog does not know about the service. It returns a scope
through `OnScopeAppliedListener`, and the caller posts the
serialized scope to the service as a
`PlayerEvent.ApplyScope` on the process bus. The service is the
single writer of the active scope, and its `rebuildPlaylists()`
method is the single mutator of the three-tier playlist. An empty
filter scope reaches the service like any other scope:
`rebuildPlaylists()` produces an empty `mScopedPlaylist`, and the
playlist view shows its empty state until the user changes the
scope. The scope is not rewritten into `ALL_SONGS` at the service
boundary, and a persisted empty filter scope round-trips through
`serialize()` and `deserialize()` unchanged.

The dialog shares the playlist's edge fast-scroll through
`EdgeScrollHelper`: a vertical drag that starts inside the list's
leftmost or rightmost 32 dp scrolls proportionally to the total
scrollable range, anchored at the position the drag began. The edge
width, engagement threshold, and scroll rate are identical on both
surfaces because both surfaces install the same helper.

### 7. The notification surface (`NotificationController`)

`NotificationController` owns the playback notification, the library
notification, both channels, the media-session metadata, and the
media-session playback state.

The controller commits through
`commitFromBroadcast(generation, snapshot)`, which
`MusicService.broadcastUpdate()` calls immediately after posting the
sticky `PlayerEvent.Snapshot` on the bus, with the exact snapshot
the event described. The player screen and the playlist activity
commit inside their sticky `PlayerEvent.Snapshot` subscribers, on
the same main-thread turn the sticky post fired, with the same
snapshot.

`broadcastUpdate()` is the only path that commits the notification's
rendered content. `snapshotForNotification()` reads
`getDisplaySong()` — the same method `broadcastUpdate()` reads to
populate the player screen — so a broadcast that fires during
`PREPARING` and the commit it triggers render the same song for the
same generation. Both surfaces advance on the same main-thread turn,
and the awareness poll that runs on the controller's cadence reads
the same value the snapshot carried. A generation filter drops any
read whose generation is strictly lower than
`mLastCommittedGeneration`, so a poll that captured state before a
snapshot cannot re-apply it after the snapshot lands. The commit
also writes the media-session playback state from
`mStateSource.sessionPlaybackState()`, so the media session, the
notification, and the player screen all agree about what is playing
at every instant.

The controller's resilience has seven parts:

1. The cancel branch — the `isIdle()` check — is refused while the
   service reports a transition pending through
   `MusicService.isTransitionPending()`.
2. The foreground promotion is attempted after `notify()`, not
   before, and a promotion failure leaves the notification visible
   and `mPlaybackForeground` false, so the next sync retries. The
   promotion decision is a two-input expression: the state is
   playing, or the service reports that an interruption is in
   progress and the notification must stay foreground. The second
   input is what keeps the notification, the shade entry, and the
   lockscreen controls alive through a phone call, when playback is
   paused but expected to resume.
3. `mLastCommittedGeneration` is advanced only when the render
   succeeds, so a failed render is retried rather than recorded as
   already committed.
4. A failed render rolls the media session back to the previous
   snapshot through `applyToMediaSession(previousSnapshot)`, so the
   notification and the media session never disagree.
5. A service command with a null intent — a sticky restart after a
   process kill — restores the notification from the persisted state
   through `MusicService.restoreNotificationFromPersistedState()`,
   so the shade is not empty after the process has been reclaimed.
6. The notification's foreground lifetime is projected from the
   service through `ServiceStateSource.shouldRemainForeground()`,
   which returns true while an interruption is in progress, the
   state is resumable, and the playlist is non-empty. The
   projection is a pure function of the interruption record, so
   the notification and the audio-focus handling cannot disagree
   about whether the interruption is still in effect.
7. A sticky `SettingsEvent.ThemeColorChanged` reaches the
   controller's `@Subscribe onThemeColorChanged` handler. The
   handler does not repaint the notification in place. The activity
   that hosts the notification recreates on the same event, and the
   notification's chrome is rebuilt from the persisted theme on the
   next commit. The themed placeholder and the themed action icons
   are keyed by the color they were built against, so a stale entry
   is never reused.

The notification's own transport buttons dispatch directly to
`MusicService.onStartCommand` through `PendingIntent.getService`
with private actions (`ACTION_PLAY_PAUSE`, `ACTION_NEXT`,
`ACTION_PREV`). `MusicService.onStartCommand` handles each
synchronously on the main thread. This is one hop — the same path
the close button uses. The media session still receives lockscreen,
Bluetooth, and headset media buttons through the unchanged
`ACTION_MEDIA_BUTTON` path; only the notification's own buttons
take the direct path.

The controller imports `com.ark.llama.event.LlamaEventBus`, not
`com.ark.llama.event.Bus`.

### 8. The Metadata Editor (`MetadataEditor`, `MetadataReader`, `MetadataWriter`)

`MetadataEditor` has two modes. In `MODE_VIEW` every field is
read-only and the primary action is MODIFY. In `MODE_EDIT` every
field is editable and the primary action is UPDATE.

A metadata read produces a `MetadataReader.MetadataBundle` whose
`status` field distinguishes `SUCCESS`, `FILE_UNREADABLE`, and
`PARSE_FAILED`; a write produces a `MetadataWriter.WriteResult` whose
`status` field enumerates every outcome the writer can produce. The
activity branches on those statuses for all control flow; the
error-message field is used only for display and logging.

**Read path.** `readMetadataSafely()` returns a bundle for every
input. The URI's write permission is checked before the attempt, so a
grant that has been revoked from Settings → App Info → Storage is
cleared rather than retried on every open; the file path is then read
directly. When the URI read is attempted and does not return
`SUCCESS`, the file path is tried. When neither source succeeds,
`loadCurrentSongInViewMode()` keeps the editor on the current song:
the title, artist, album, and genre fields are populated from the
song database's row for the path, the header stays on VIEW MODE, and
MODIFY is enabled so the user can re-establish file access through
the storage access framework. `onCreate()` wraps
`Song.getContentUri()` and `CharacterMapper.getSongFromFile()` in
try/catch, so a malformed cached URI or a file whose embedded tags
cannot be read resolves to a null field rather than to an exception.
The editor always opens on the song `MusicService.getCurrentSong()`
names, whose playlist entry is refreshed by
`MusicService.refreshMetadataForSong` when the
`MetadataEvent.Updated` event lands.

**URI and path.** The editor treats the URI the current session
selected as its write target, and the absolute path the song database
and the playback consumers key on is always paired with that URI. When
the editor is opened with a `song_uri` extra, a supplied `song_path`
is accepted only when `getFilePathFromUri(uri)` resolves the URI to
it, or when `SongDatabase.getSong(suppliedPath)` returns a row whose
`getUri()` equals the supplied URI; otherwise `mFilePath` stays null.
When the editor is opened with only a `song_path`, the URI is read
from the database. The SAF picker result augments identity rather
than replacing it: the pick adds the URI, and a resolved path refines
the path but never removes one the caller already supplied. The URI
is authoritative for writing; the path is authoritative for matching
receivers. MODIFY prefers the session-selected URI over a stored URI
when the session has one with write permission, and persists it
against the current path through `saveUriToDatabase()` before
entering edit mode.

**Save.** `performSaveChanges()` captures the target URI, the target
path, the album-art-removed flag, the pre-edit bundle, and the
submitted update into `final` locals before it dispatches the write
to the executor, and passes them to the success and failure handlers
as parameters. A later SAF pick or a later change to `mGrantedFileUri`
or `mFilePath` cannot redirect a save that is already running.

`onSaveSuccess()` builds the announcement from the submitted update
overlaid on the pre-edit bundle through
`buildAnnouncementFromSubmitted`, and the post-write re-read is
verification only: a re-read that returns `FILE_UNREADABLE` or
`PARSE_FAILED` logs a warning, and the submitted values are announced
anyway, because the write itself succeeded. A re-read failure cannot
suppress the event, the database row, or playback resumption. The
re-read, when it succeeds, becomes `mOriginalData` — the baseline the
next edit's change detection compares against — but it does not
become the announced values. The shared album-art cache is evicted
under both the path key and the URI key before the event is posted.
The event always carries the saved URI; the path is included when
the editor knows one.

`onSaveFailure()` takes the captured target path as a parameter and,
on `PERMISSION_DENIED`, writes the fallback database row against that
path instead of `mFilePath`.

**Consumers.** The `MetadataEvent.Updated` event has three
subscribers. `MusicPlayer.onMetadataUpdated` resolves the edited
song's path through `resolveSongPathFromMetadata`, calls
`MusicService.refreshMetadataForSong`, and invalidates the carousel's
per-path cache through `AlbumArtCarousel.invalidatePath`.
`PlaylistActivity.onMetadataUpdated` resolves the same path and
updates the display row, refreshing the playing indicator when the
edited song is current. `MusicService.onMetadataUpdated` is a bus
subscriber registered in `onCreate` and unregistered in `onDestroy`,
so an edit lands on the in-memory playlist even when the player
activity has been reclaimed but playback continues. On a SAF-only
write, the editor knows the URI but not the path; each consumer's
`resolveSongPathFromMetadata` helper resolves the URI against
`SongDatabase` so path-keyed consumers still find the edited song.

### 9. The overlay stack (`OverlayManager` and `ToastManager`)

`OverlayManager` is the process-wide, reason-keyed dim-and-spinner
overlay. It is a singleton; the spinner and the dim background belong
to the process, not to any single activity. When the activity in
front changes, the overlay is reattached to the new activity's
content view without the reason set being touched.

`ToastManager` is the operation-stack manager for persistent overlays
and temporary toasts. An operation is registered on the stack by its
ID; the stack renders the top operation and retains the lower ones.
Cancelling an operation is removing its entry from the stack, which
is indistinguishable from an operation that completed. Callers that
need to distinguish the two carry their own terminal state, as
`PlaylistManager` does for its genre load and its reset.

Temporary toasts are owned by `ToastManager`, not by any operation.
They auto-dismiss after a fixed duration and are never reattached.
A temporary toast coexists with a persistent overlay; showing one
does not transition the state machine away from
`SHOWING_PERSISTENT`.

A toast's vertical position is chosen from the window's current
height, through `resolveBottomMarginDp()`. The manager holds two
margins — `TOAST_BOTTOM_MARGIN_DP = 76` and
`TOAST_BOTTOM_MARGIN_COMPACT_DP = 68` — and picks the compact one
below `COMPACT_LAYOUT_HEIGHT_DP = 400`. So a toast is centred in
full-screen and in split-screen alike, in every activity, without
any activity having to announce its own layout state.

A toast's horizontal span is read from the content view's measured
width through `resolveAvailableToastWidthPx(ViewGroup)`, floored at
zero and reduced by `TOAST_HORIZONTAL_INSET_DP = 16` on each side.
The toast's own text sits with `TOAST_PADDING_HORIZONTAL_DP = 8`
inside that frame, so a compact floating label reads as a label
rather than as a padded content card. The toast is single-line and
ellipsizes only when the message genuinely exceeds the available
span. A persistent overlay's message is re-read from
`resolveAvailableToastWidthPx(ViewGroup)` on every render, so a
message that grows past the width the overlay was created with
ellipsizes at the current window's usable width.

The overlay's visible text is composed by `ToastManager` alone,
inside `composeOverlayText(title, body)`, from the action verb and
the count that the operation's owner passes as two separate strings,
joined with a middle dot. The notification mirror receives the same
two strings separately, through
`ToastManager.showPersistentOverlay(Activity, String, String, String)`
and `ToastManager.updatePersistentOverlay(String, String, String)`.
Neither surface splits a joined message, and no resource carries a
separator. No caller composes the joined form; the composition lives
at the single site in `ToastManager` that writes the overlay's text.

A sticky `SettingsEvent.ThemeColorChanged` reaches `ToastManager`'s
`@Subscribe onThemeColorChanged` handler, which tears down every
persistent overlay's view and re-creates it through the same
`createOverlayInView` path the initial show uses, so the current
theme color is read at construction rather than patched in place.
Temporary toasts are not rebuilt; their lifetime is shorter than a
rebuild would require, and the next temporary toast is constructed
against the new color anyway.

Toast durations are `long` values in milliseconds. `LENGTH_SHORT`,
`LENGTH_NORMAL`, and `LENGTH_LONG` are `long` constants on
`ToastManager`, and `showToast` and `showToastInView` take a `long`
`duration`. Every caller that passes an integer literal widens
implicitly.

### 10. The album-art carousel (`AlbumArtCarousel`)

The carousel is a pure renderer of the window the service publishes.
A three-slot strip three slot-widths wide sits inside a viewport one
slot wide; the strip's middle slot is aligned to the viewport at
rest. A horizontal drag follows the finger, and on release the strip
either commits to a neighbour or settles back to rest.

Every other transition — a transport button, a headset key, an
automatic advance, an incoming file — is animated from the service's
`PlayerEvent.Snapshot` event alone. The activity forwards the
window, the direction, and the restart the service named to
`setWindow`, and the carousel renders exactly the animation they
describe.

- **Commit.** A commit presets the confirmed window before the slide
  starts, so the animation's end action adopts the window
  synchronously and the strip never enters the
  snapshot-awaiting state.
- **Restart.** A restart adopts the incoming window from the first
  frame and runs the below-threshold nudge. Both legs of the nudge
  run on the same spring constants — the outward leg carries the
  strip to the displaced position and the return leg carries it back
  over the same distance, on the same damping ratio and stiffness —
  so the two halves of the gesture take the same time to settle and
  overshoot by the same small amount.
- **Reveal.** A non-neighbour change runs the cross-fade: the
  outgoing art fades out on the emphasized accelerate curve and the
  incoming art fades in on the emphasized decelerate curve with a
  subtle scale-up, over the same total duration a commit takes. The
  cross-fade waits for the incoming art to resolve before it begins,
  so the outgoing art stays on screen while the incoming art is being
  decoded and the transition the user sees is one cross-fade from
  the outgoing art to the incoming art, never a placeholder in
  between.

Every motion toward a rest position — the nudge's two legs, a
returned swipe, a refused press, a snapshot timeout — runs on the
same spring, so the strip's physical settle is the same wherever the
motion began.

Every animation ends in a single main-thread turn that renders the
carousel's new bitmaps and hands the same turn to the activity's
commit runnable, so the album art, the title, the artist, the seek
bar, the transport glyph, the notification, and the media session
land on one frame.

### 11. The edge fast-scroll (`EdgeScrollHelper`)

The helper installs an `OnItemTouchListener` on a `RecyclerView` that
claims a vertical drag once the finger has crossed the touch slop
inside the list's leftmost or rightmost `EDGE_ZONE_DP = 32`. The drag
then maps linearly onto the total scrollable range: a full
viewport-height of finger travel moves through
`FAST_SCROLL_FRACTION_PER_VIEWPORT = 5%` of the range.

The gesture engages only when the content is at least
`FAST_SCROLL_MIN_CONTENT_RATIO = 3×` taller than the viewport; on a
shorter list the edge zones defer to the `RecyclerView`'s own
scrolling. A tap is never consumed.

The playlist and the playback-scope dialog both install it, so the
edge width, engagement threshold, and scroll rate are the same on
both surfaces.

### 12. The storage observer (`StorageObserver`)

`StorageObserver` registers a single content observer, for
`MediaStore.Audio.Media.EXTERNAL_CONTENT_URI`. A change notification
starts a quiet period of `QUIET_PERIOD_MS = 5000`; if no further
change arrives inside the period, the observer asks `PlaylistManager`
to run a scan through `updatePlaylistSilent()`, then posts
`PlaylistEvent.Refresh` and (after a short delay)
`PlaylistEvent.TriggerGenreLoading` on the bus through
`LlamaEventBus.post`.

`handleChange` filters the notification's URI for the `/audio/` path
segment, so a notification that arrives with an unexpected URI is not
acted on. `selfChange` is not consulted: audio metadata edits
performed by this process are flagged as self changes and must
refresh the playlist.

A cooldown of `SYNC_COOLDOWN_MS = 60000` prevents the scan from
running more than once per minute.

### 13. The in-memory song list (`MusicLibrary`)

`MusicLibrary` owns the in-memory song list. `refreshCache()` is its
sole writer: it reads the songs table and the loose-songs table from
`SongDatabase` — the authoritative store — and rebuilds the list under
a `ReentrantReadWriteLock`.

Multiple `MusicLibrary` instances may coexist, one per component that
reads song data. Each instance holds its own in-memory state, so each
must refresh independently when notified. The database keeps them in
agreement: a `PlaylistEvent.Refresh` on the bus reaches every
subscribed instance, and every instance converges on the same
database state.

`invalidate()` is the reset-path counterpart to `refreshCache()`. Where
`refreshCache()` re-reads the song and loose tables while keeping the
current folder set, `invalidate()` additionally clears the in-memory
lists, drops the loaded flag, and re-reads the folder URI set from
`SharedPreferences`. Every `MusicLibrary` instance in the process
converges on the cleared selection, so a subsequent scan of the
removed folders cannot repopulate the database.

`MusicLibrary.postSongAdded(song, currentCount)` posts
`PlaylistEvent.SongAdded` once per song during a progress-reporting
scan. The event carries the seven fields the playlist row needs; the
playlist activity appends each event to its display list
incrementally.

`MusicLibrary` exposes both a
`val songCount: Int get() = getSongCount()` property and a
`fun getSongCount(): Int` method, so a caller that wants the count
without a defensive copy can read `library.songCount` rather than
paying the copy cost of `getAllSongs().size`. The two read the same
number through two different methods; the difference is that
`getAllSongs()` returns `ArrayList(mAllSongs)` — a copy — while
`getSongCount()` returns the size of the in-memory list without
copying it.

### 14. The playlist search surface (`PlaylistActivity` and `PlaylistAdapter`)

The search bar at the top of the playlist filters the display list
on every keystroke. The query is lowercased once and matched with
`String.contains` against the lowercased title, artist, album, and
genre of every song in the current playlist, so a query matches a
substring of any of the four fields and the match is case-
insensitive in every script. The four fields are the same four the
service's own scope matcher reads, so a search and a genre scope
describe the same underlying metadata.

A song with a null field is skipped for that field; the null is not
treated as an empty string that matches every query, because a query
of `"a"` should not surface every song whose artist is absent.

The filtered result is sorted by the same comparator the full list
uses, through `applyCurrentSort(false)`, so the sort mode the user
last chose continues to govern the order of the results. The sort
button is hidden while a query is active: a re-sort of the filtered
results would change their order without telling the user that the
underlying playlist had been re-sorted, and the visual result would
be a list that reorders on a button tap that already had no effect
on the query.

The header shows the visible count over the total playlist count
through `song_count` plurals, so the plural agreement follows the
total the label is about. When the query is cleared, the header
returns to the sort-mode label and the count line returns to the
total.

The search bar's visibility is separate from the playlist's empty
state. When the playlist is empty, the search button is disabled
and the search bar is hidden; when the playlist is non-empty, the
search button is enabled and shows the bar. Clearing the query
through the clear button, the IME action, or a back press restores
the full list, and each of those paths runs the same
`performSearch("")` that the text watcher runs on an empty query, so
the three exits agree on the result.

`PlaylistActivity`'s display list is a Kotlin `ArrayList<Song>`, which
implements Kotlin's `List<Song>`; passing it to
`PlaylistAdapter.updateList(newSongs: List<Song>)` requires no cast.
`PlaylistAdapter.FavoriteListener` is a regular Kotlin interface, not
a `fun interface`, so `setFavoriteListener` is called with
`object : PlaylistAdapter.FavoriteListener { override fun
onFavoriteToggle(songPath: String, favorite: Boolean) { ... } }` and
not with a lambda.

### 15. Split-screen behaviour (per activity)

Every activity declares the same `configChanges` set and adjusts its
own content when the window resizes, instead of being recreated.
There is no `SplitScreenBehaviour` class; the behaviour lives in the
activities that own the affected layouts.

- `MusicPlayer` sizes, shrinks, or hides the album art. When the album
  art hides, the wrapper around it is hidden with it, the
  title-and-artist group is pinned to the top of its weight slot at
  the album art top's Y offset so the title lands where the album art
  used to begin, and the transport row's bottom padding is reduced so
  the group's slot has room for the artist line. Every view below the
  group keeps its position relative to the window bottom because the
  group's weight slot absorbs the difference.
- `EqualizerActivity` toggles the band wrapper's layout weight so the
  band controls take the space the effect controls leave in the full
  layout, and take their natural height in split mode — the band
  controls and the effect controls share one scroll container, so in
  split mode the two sections scroll together.
- `FolderManager` keeps the list visible in every window size and
  shows its placeholder only when the list is empty. The SCOPE
  button in the header stays available in split mode; a tap on it
  is debounced at 500 ms, so a rapid double-tap opens one dialog,
  not two.
- `MetadataEditor` and `PlaylistActivity` let their existing scroll
  containers absorb the overflow.

### 16. The Home button gesture (`MusicPlayer`)

The Home control on the main player screen is a two-tap
confirmation, with three visible states carried by a three-value
`HomeConfirmState` enum — `IDLE`, `ARMED`, `CONFIRMED`.

`IDLE` is the steady state, painted in the inactive control
colour every action button uses when it is enabled but not
expressing an active mode. A first tap enters `ARMED`: the
button is repainted in the themed active colour, an arming
toast is shown, and a timeout at `HOME_CONFIRM_WINDOW_MS`
reverts the button to `IDLE` if no second tap arrives. A second
tap inside that window enters `CONFIRMED`: the button is
repainted in the pending colour and the return-home is scheduled
at `HOME_CONFIRM_DWELL_MS` later, so the user perceives the
confirmation as a colour rather than as an intent that fires
before the screen updates. The dwell's body calls
`moveTaskToBack(true)`, which asks the ActivityManager to move
the task behind the next one in the stack — the same mechanism
the framework uses for back-at-root, and one that does not
depend on resolving a home intent.

Both phases are carried by a single `mHomeReturnRunnable`
field; its body branches on `mHomeConfirmState` to decide
whether it is ending the ARMED window or the CONFIRMED dwell.
`cancelReturnHome()` cancels the pending runnable without
needing to know which phase scheduled it, resets the state to
`IDLE`, and repaints the button in its inactive colour. Every
transport button, every navigation button, every committed
album-art gesture, `onStop`, and `onDestroy` call
`cancelReturnHome()`, so the user's most recent tap — whether
on the Home button or on any other control — is always the
one that decides what happens next.

The activity's own helpers follow one shape: the verb, the
object, and `Safely` trailing. `updateUISafely(Runnable)`
coalesces UI updates that arrive while another is in flight
and dispatches them on the main thread. `callServiceSafely(Runnable)`
holds the service lock and dispatches only when the service is
bound. `recycleBitmapSafely(Bitmap)` skips a bitmap that is
already recycled and swallows the recycle exception. The two
receiver helpers — `registerReceiverSafely` and
`unregisterReceiverSafely` — have followed the same shape
since the receiver set was consolidated. The consistency is
the point: every helper that absorbs a failure the caller does
not have to handle reads the same way in every call site.

### 17. Cross-cutting: theme and language rebuild

Two sticky settings events — `SettingsEvent.ThemeColorChanged`
and `SettingsEvent.LanguageChanged` — share a single contract
across every surface of the app.

`ThemeManager.setThemeColor(int)` and
`LanguageManager.setLanguage(String)` each persist the new value and
post a sticky event on the process bus through
`LlamaEventBus.postSticky`. Neither touches a view. Every activity
subscribes in STICKY mode and answers each with
`recreateWithConfiguration()`. Every dialog that subscribes to
`SettingsEvent.ThemeColorChanged` (`CreditsDialog`) dismisses
itself; the three dialogs that do not subscribe to it
(`LanguageDialog`, `PresetDialog`, `PlaybackScopeDialog`) are
dismissed transitively by their hosts' recreation. `ThemeDialog`
reports the selection through its listener, and the caller's
listener persists it; the dialog does not call `setThemeColor()`
itself. `ToastManager` tears down and re-creates its persistent
overlay's text view through the same `createOverlayInView` path its
first show uses. `NotificationController` does not repaint in
place; its themed placeholder and action icon caches are keyed by
the color they were built against, so the recreated activity's
next commit reads the persisted theme and either reuses a matching
entry or rebuilds.

The invariant is the same for both events: **the event is a
signal to rebuild, not an instruction to patch.** There is no view
in the app whose theme or locale is written after the frame it was
first drawn on. Every color and every string visible to the user is
the result of a freshly constructed view or a freshly built
notification reading a persisted value. The two paths cannot drift
apart on any device because they run the same code shape.

The same reasoning extended to the language picker in 1.3.0 and to
the theme picker in 1.4.6. The `LanguageManager` and `ThemeManager`
classes carry the full contract in their class KDoc.

### 18. Cross-cutting: localization

Every user-visible string the app shows resolves from
`res/values/strings.xml` in the base locale and from a
`values-XX/strings.xml` in each of the fifteen additional locales.
`LanguageManager` owns the selected language tag, the supported-
language set, and the `wrap(Context)` helper that produces a context
whose configuration carries the selected locale. Every activity
overrides `attachBaseContext(Context)` and calls `wrap` before any
state exists, so the framework reads from the matching resource
folder for the whole of the activity's lifetime. A language change
is a persisted tag, a sticky `SettingsEvent.LanguageChanged` on the
bus, and a `recreate()` on each activity that subscribes; no restart
of the process is needed.

The supported set is declared three times —
`LanguageManager.LANGUAGE_TAGS`, `res/xml/locales_config.xml`, and the
`resourceConfigurations` block in `app/build.gradle.kts` — and the
three lists describe the same set. Sixteen languages total: English
plus fifteen additional locales.

Every one of those strings is a resource, including the unit symbol
that appears on the equalizer's per-band frequency labels. Those
labels read `equalizer_frequency_hz`, `equalizer_frequency_khz`, and
`equalizer_frequency_khz_decimal`, and each locale supplies the unit
its language uses — `Hz`/`kHz` in the Latin-script locales,
`Гц`/`кГц` in Russian. The decimal separator between a kilohertz
value and its tenths is not in the resource: it comes from the JVM
default locale, which `LanguageManager.wrap` has already set to the
app's selected language, so the same string renders as `1.2 kHz` in
English and `1,2 kHz` in German.

The zero-value strings for semitones and decibels are parameterized
for the same reason. `equalizer_semitones_zero` is `%1$.1f st` and
`equalizer_db_zero` is `%1$.1f dB`, so the zero value is formatted by
the same code path that formats the positive and negative variants,
and the decimal separator follows the app's language.

**The album-art cache is process-scoped.** `AlbumArtCache` is a
singleton that outlives every activity, and both `MusicPlayer` and
`NotificationController` hold a reference to its underlying
`LruCache` rather than owning a cache of their own. A configuration
change or a theme change does not evict the bitmaps, and the
carousel's first read after the recreate finds the entry the previous
instance decoded. Under sustained memory pressure —
`MusicPlayer.onTrimMemory` at `TRIM_MEMORY_BACKGROUND` or higher —
the cache is evicted wholesale, and `MusicService.onTrimMemory` at
the same level evicts the notification art cache.

The top bar in every activity and every dialog lays its title out in a
horizontal linear layout with a weight of 1, so the title is the only
element that can yield. A translation longer than the space the two
edge controls leave truncates with an ellipsis rather than running
under a button beside it. The playback-scope dialog's title is one of
the titles that follows this rule; a long `SCOPE MODE: <label>` at a
narrow width truncates before it reaches the CANCEL or APPLY label.
The same reasoning is why the separator between the sort-mode prefix
and the mode label is a Java-side constant rather than a trailing
space on the prefix resource: the resource compiler strips trailing
whitespace from a non-formatted string, and a translator cannot
preserve a space they cannot see.

Layout spacing uses explicit 8dp margins rather than string-based
separators in headers and list rows, ensuring a visible, consistent
gap across all screen sizes and languages. In right-to-left languages
like Arabic, the Music Player timers, the Credits version header, and
the dynamically generated Equalizer frequency bands use relative
`START` and `END` gravity to anchor perfectly to the outer 16dp edges
of their containers, ensuring a balanced and symmetrical UI in every
language. The Metadata Editor's form uses explicit
`layout_marginTop` values to guarantee uniform bounding boxes for
every field label.

A language change closes the surface the choice was made from and
recreates the activity in one step. The dialogs do not rebuild
themselves in place. Every surface the user was on when the choice
was made is dismissed, and the activity is rebuilt with a wrapped
base context that carries the new locale. The next time a dialog is
opened it is constructed against the new locale from the start.

The theme dialog's row and the preset dialog's row share one shape.
The row's own padding is symmetric and logical, so both its edges
carry the inset in every direction. Each child that sits inside the
row — the name, the colour preview circle, the `CURRENT` indicator —
carries at most a single logical start margin, which describes the
gap between it and its sibling. The trailing inset on whichever child
lands at the trailing edge is supplied by the row's own end padding,
not by a second margin on the child. That gives the row one writer
for each edge and one writer for each gap, which is what keeps the
layout correct under LTR and RTL without any special-casing.

</details>

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