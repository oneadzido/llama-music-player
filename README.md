# Llama Music Player

A high-quality Android music player with a custom playback engine, pitch and playback speed control, a sixty-preset equalizer, metadata editing capabilities, favorites, playback scoping, and a sixteen-language interface.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Android](https://img.shields.io/badge/Android-6.0%2B-brightgreen.svg)](https://developer.android.com)
[![Privacy](https://img.shields.io/badge/Privacy-Policy-blue.svg)](PRIVACY.md)

[![Download Latest Release APK](https://img.shields.io/badge/Download-Latest_Release_APK-brightgreen?style=for-the-badge&logo=android&logoColor=white)](https://github.com/oneadzido/llama-music-player/releases/latest/download/LlamaMusicPlayer-1.4.0.apk)

[![Build](https://github.com/oneadzido/llama-music-player/actions/workflows/main.yml/badge.svg)](https://github.com/oneadzido/llama-music-player/actions/workflows/main.yml)

---

## Origin Story

Llama Music Player was created by a self-taught, AI-assisted software developer from Ghana.

Driven by a lifelong passion for computers and music, he began building on an Android smartphone, without a laptop or formal programming training. Working alongside AI, he turned a simple idea into a feature-rich Android music player through experimentation, hands-on testing, problem-solving, and continuous improvement.

His philosophy, The Llama Way, is simple: keep the experience intuitive, avoid unnecessary complications, and respect the listener's privacy.

He built with the tools he had, learned along the way, and turned his vision into reality.

Read the full origin story here: [ORIGIN_STORY.md](ORIGIN_STORY.md)

---

## Download

[Download Latest Release APK](https://github.com/oneadzido/llama-music-player/releases/latest/download/LlamaMusicPlayer-1.4.0.apk)

Or browse the release page:

- [GitHub Releases](https://github.com/oneadzido/llama-music-player/releases/latest)
- GitHub Actions artifacts

---

## Features

- Audio Formats: MP3, FLAC, OGG, M4A, AAC, WAV
- Pitch Control: -6.0 to +6.0 semitones
- Playback Speed Control: 0.50x to 1.50x
- 10-Band Equalizer with 60 presets including Custom
- Bass Boost & Surround: 0-100% adjustment
- Sixteen Languages: English, Español, Français, Deutsch, Italiano, Português, Русский, 中文, 日本語, 한국어, हिन्दी, العربية, Türkçe, Bahasa Indonesia, Nederlands, Polski. The picker lives in the Credits dialog and rebuilds the UI in the new locale without a restart
- Favorites: Double-tap the album art, tap the heart beside the transport controls, or tap a playlist row's heart to mark a song. Playlist heart taps stage in memory and commit in one batch when the playlist screen pauses
- Playback Scope: Restrict the player to favorites, loose tracks, or a chosen set of folders, genres, artists, or albums. The dialog shows a live count of the songs the selection would yield. The playing song and any pending next stay in the queue regardless of scope
- Exit Button: The main player screen's exit button closes the app with a deliberate two-tap gesture. The first tap turns the button red and shows a confirmation toast; the second tap within one second exits. If the second tap does not arrive, the button reverts to its normal color
- Centre Swipe: Swipe up on the centre of the album art to resume a paused song, or down to pause a playing one. The gesture requires twice the touch slop of travel to distinguish it from digitizer wobble, and commits on release rather than on each move, so a single drag toggles at most once
- Album-Art Carousel: Swipe left or right across the album art, or tap the transport buttons, to move between songs. Files opened from other apps transition through a cross-fade
- Volume Gesture: Swipe up or down along the album-art edge to change the music volume
- Volume-aware Playback: Music pauses at volume 0 and resumes when the volume is raised
- Auto-Resume After Transient Focus Loss: A phone call or navigation prompt pauses the music; when focus returns, playback resumes from where it paused. A permanent loss is respected
- Automatic Ducking: A short interruption lowers the music's own volume for its duration instead of pausing
- Playlist & Scope Fast-Scroll: Drag vertically along the leftmost or rightmost edge of a long list to scroll proportionally. The playlist and the playback-scope dialog share the same implementation, so the edge width, engagement threshold, and scroll rate are identical on both surfaces
- Sort Progress Feedback: Cycling the sort mode shows the dim-and-spinner overlay, hidden only once the reordered list has settled and drawn
- Library Load Feedback: A persistent overlay appears while the library loads on launch
- Metadata Editor: Edit ID3 tags, lyrics, and album art. A write lands on the file the user is editing, whatever the entry path, and the post-write reread is verification only
- Folder Management: SAF folder import; removing a folder leaves every other folder's songs and every loose track alone. Each imported folder carries a badge identifying its storage medium
- Live Search: Real-time filtering
- 10 Sort Modes: Title A-Z, Title Z-A, Artist A-Z, Artist Z-A, Album A-Z, Album Z-A, Genre A-Z, Genre Z-A, Date (Newest First), Date (Oldest First)
- 32 Theme Colors
- Split-screen Layout: The album art resizes, shrinks, or hides with the window height; every screen adjusts its own content so the controls always fit
- Bluetooth Support: Headset controls
- Background Playback: Persistent notification with album art
- Lock Screen Controls: MediaSession integration

## Technical Details

- Minimum SDK: Android 6.0 (API 23)
- Target SDK: Android 15 (API 35)
- Build Tools: AGP 8.2.2, Gradle 8.2, JDK 17
- Metadata Parsing: jaudiotagger 3.0.1
- Icons: FontAwesome 6 (Free Solid)

### Permissions

Android asks for sensitive permissions when the feature that needs them is first used. The app declares the permissions it needs in the manifest and requests the runtime grants at the appropriate point in the experience.

- `READ_MEDIA_AUDIO` (Android 13+) / `READ_EXTERNAL_STORAGE` (Android 6.0–12) for music-library access.
- `POST_NOTIFICATIONS` (Android 13+) for playback and library notifications.
- `BLUETOOTH_CONNECT` (Android 12+) for headset connection state and media-button routing.
- `READ_MEDIA_IMAGES` (Android 13+) / `READ_EXTERNAL_STORAGE` (Android 6.0–12) when the Metadata Editor needs to read an image selected for album artwork.
- `FOREGROUND_SERVICE` and `FOREGROUND_SERVICE_MEDIA_PLAYBACK` are manifest permissions used to run the media-playback foreground service. The media-playback permission is a normal permission granted at install time; Android does not present it as a runtime permission.
- `WAKE_LOCK` and `MODIFY_AUDIO_SETTINGS` support uninterrupted playback and audio effects. Battery-optimization exemption is offered through Android's system settings when the user chooses that option.

## How It Works

<details>
<summary><b>Inside the codebase — ten subsystems and one cross-cutting concern worth knowing about</b></summary>

Ten subsystems are worth knowing about when reading the source, and one cross-cutting concern touches every activity.

**The playback state machine** lives in `MusicService`. A single
`ServiceState` enum, two monotonic counters, and a two-index model
(`mCurrentIndex` / `mTargetIndex`) hold every transition-related value.
State mutations run on the main thread; raw `MediaPlayer` calls run on
a dedicated `HandlerThread`. Only marshalled `Runnable`s cross between
them. The `READY` state is the state a song sits in when it has been
prepared and is waiting for the play button.

Six rules belong to this machine and are worth stating explicitly.
First, the music-stream volume receiver is a facet of it: registered in
`onBind(Intent)` and unregistered in `onDestroy()`, so once the user
has opened the player a drop to zero pauses and a raise above zero
resumes or starts. Second, a pause caused by a transient audio-focus
loss is remembered; when the framework returns `AUDIOFOCUS_GAIN` and
the machine is still paused, playback resumes from the paused
position. A permanent loss is not remembered: the user has moved on,
and playback does not restart on its own. Third, a duck request —
`AUDIOFOCUS_LOSS_TRANSIENT_CAN_DUCK`, what a navigation prompt or a
notification tone sends — does not pause. The machine lowers the
MediaPlayer's own output volume to `DUCK_VOLUME_FRACTION` for the
duration of the interruption and restores it to full on
`AUDIOFOCUS_GAIN`. The system stream volume is not touched, so the
volume slider does not move and the receiver is not involved. Fourth,
volume-up at the device maximum does not emit `VOLUME_CHANGED_ACTION`
— no value changed — so `MusicPlayer` overrides `dispatchKeyEvent` to
resume a paused song from the foreground when the volume is already at
max. Fifth, the foreground path is the only one the activity can offer
for that case. The framework does not deliver volume keys to a
backgrounded activity, and no broadcast fires at the maximum because
no value changes. When the player is backgrounded and the song is
deliberately paused, resuming goes through the notification's play
action, which reaches the service through the media session in every
screen state. Sixth, every transition the machine begins names its
direction and its restart as properties of the generation that
produces it. A next or previous request notes `+1` or `-1`; an
automatic advance notes `+1`; a reprepare of the same song notes `0`;
a wrap that restarts the current song sets the restart flag. The
values are consumed into the generation when the transition starts
and published alongside the window on the same broadcast, so the
carousel's animation and the audio's transition are the same value by
construction. A headset disconnect is a seventh facet of the same
machine: the service registers a receiver for the pause request in
`onCreate` and unregisters it in `onDestroy`, so the pause fires
whether or not the player activity is alive.

The same machine also owns the three-tier playlist. `mFullPlaylist`
is the immutable database reflection, `mScopedPlaylist` is
`mFullPlaylist` filtered by the active `PlaybackScope`, and
`mCurrentPlaylist` is the active queue, which is `mScopedPlaylist`
plus the songs at `mCurrentIndex` and `mTargetIndex` — the grace
period that keeps a currently-playing song and its pending next
alive regardless of scope. `rebuildPlaylists()` is the single
mutator for the latter two. A `PlaybackScope` is a value object with
seven kinds — All, Favorites, Loose Tracks, Folders, Genres, Artists,
Albums — serialized to preferences as JSON through the framework's
`org.json` package. `commitFavorites(Map)` writes the per-song
favorite bit through `SongDatabase.setFavorite(String, boolean)` and
rebuilds the playlists from it. The favorite column is written only
by that targeted path and by the fresh-row branch of the insert;
every other metadata write goes through
`SongDatabase.updateSongMetadata(Song)`, which never touches the
column, so a scan or a genre load cannot clobber a user's favorites.

Another entry point into the same machine is the incoming-file
pathway. A user who opens an audio file from another app hands the
path to the activity, and the activity hands it to the bound service
through a direct main-thread method call — no broadcast, no delivery
window. The service stores the path as its newest incoming-file
request and re-enters reconciliation. A path that names a song
already in the playlist transitions to it immediately. A path that
names a song not yet in the playlist stays parked until the insert's
rebuild adds it, and the reconcile that follows the rebuild resolves
it. The slot is cleared only when the transition it names begins, so
a request that arrives while another transition is preparing
survives the prepare and plays after it settles. A path the insert
rejected is cleared by the insert's failure callback before the next
reconcile runs, so a reconcile cannot resolve to a song the playlist
does not contain.

**The album-art gesture classifier** lives in `MusicPlayer`. A single
touch on the album-art container is classified on its first
significant move into one of five states: `TOUCHED` (no significant
move yet), `CAROUSEL` (horizontal swipe), `VOLUME` (vertical swipe
that started in the left or right edge zone), `PLAYBACK` (vertical
swipe that started at the centre), and `CANCELLED` (edge-zone swipe
on a device with no music stream). The classifier owns all four
gestures the album art supports — the carousel swipe, the volume
drag, the centre swipe to play or pause, and the double-tap
favourite — and the four share one state enum so a touch cannot fall
into two modes at once. The `PLAYBACK` gesture requires twice the
touch slop of travel to distinguish an intentional swipe from
digitizer wobble on a resting finger, and it commits on release, not
on the first significant move, so a single drag toggles at most once.
The `TOUCHED` state is the resting state from which the double-tap
favourite is detected: a touch that never left the touch-slop radius
is a tap, and two taps within `ViewConfiguration.getDoubleTapTimeout()`
toggle the current song's favourite. The album-art container owns
every gesture that operates on the album art; the carousel, the
volume stream, and the service are the destinations the classifier
dispatches to.

**The playback window** is the single source of truth for the album
art. `PlaybackWindow` is an immutable value holding three paths — the
previous neighbour, the current song, the next neighbour — and the
service's monotonic transition counter. `MusicService.getWindowSnapshot()`
is the sole producer; it computes the window from two pure functions,
`peekNextIndex(int)` and `peekPreviousIndex(int)`, that are the same
primitives the service's own next and previous requests commit from.
Under shuffle, the two peek functions hold independent memos, so the
answer the window advertises in each direction survives a window read
and is the index the matching commit will play, in both the
history-backed and the fallback case. The window itself is memoised
for the current state, so two consecutive reads with the same state
return the same three paths: the neighbour slot the carousel renders
and the target the next commit advances to are the same value by
construction. The peek memos, the shuffle history, and the window
cache are index-keyed and are cleared on every operation that changes
what an index means. The carousel is the sole consumer; it holds no
playlist, no index, and no bitmap, and it adopts every window through
a single method. Alongside the window, the broadcast carries the
transition's direction (`+1`, `-1`, or `0`) and its restart flag. The
direction and the restart are properties of the generation the window
carries, so a receiver that has the window has the transition that
produced it.

**The playback scope** lives in `PlaybackScope` and
`PlaybackScopeDialog`. `PlaybackScope` is an immutable value object
with seven kinds — All, Favorites, Loose Tracks, Folders, Genres,
Artists, Albums — serialized to preferences as JSON. The dialog
presents the seven modes as a single flat list, expands a mode's
filter list below its row when the mode is selected, and computes
the song count the selection would yield from the same matcher the
service uses. The dialog does not know about the service; it
returns a scope through `OnScopeAppliedListener`, and the caller
broadcasts the serialized scope to the service through
`APPLY_PLAYBACK_SCOPE`. The service is the single writer of the
active scope, and its `rebuildPlaylists()` method is the single
mutator of the three-tier playlist.

**The notification surface** lives in `NotificationController`. It
owns the playback notification, the library notification, both
channels, the media session metadata, and the media session playback
state. The controller commits through
`NotificationController.commitFromBroadcast(generation, snapshot)`,
which `MusicService.broadcastUpdate` calls immediately after
`sendBroadcast` returns, with the exact snapshot the broadcast
described. The player screen commits inside its `UPDATE_PLAYER`
receiver, which runs synchronously inside that same `sendBroadcast`
call, so both commits land on the same main-thread turn with the
same snapshot. `broadcastUpdate` is the only path that commits the
notification's rendered content. During PREPARING,
`snapshotForNotification()` returns the current-committed song — the
song the player screen is still showing — not the target, so the
poll that runs on the controller's cadence cannot advance the
notification ahead of the player. Both surfaces keep showing the
song the previous transition committed until the transition commits,
and both advance on the same turn. A generation filter drops any
read whose generation is strictly lower than the last committed
generation, so a poll that captured state before a broadcast cannot
re-apply it after the broadcast lands. The poll remains in place as
the safety-net cadence for position updates and for state changes
that do not reach the notification through a broadcast — art
resolving after the commit, the theme changing. The commit also
writes the media session playback state from the state machine's own
destination, so the media session, the notification, and the player
screen all agree about what is playing at every instant.

**The Metadata Editor** lives in `MetadataEditor`. The activity has
two modes. In VIEW MODE every field is read-only and the primary
action is MODIFY. In EDIT MODE every field is editable and the
primary action is UPDATE. A metadata read produces a
`MetadataReader.MetadataBundle` whose `status` field distinguishes
`SUCCESS`, `FILE_UNREADABLE`, and `PARSE_FAILED`; a write produces a
`MetadataWriter.WriteResult` whose `status` field enumerates every
outcome the writer can produce. The activity branches on those
statuses for all control flow; the error-message field is used only
for display and logging.

The read path is performed by `readMetadataSafely()`, which returns
a bundle for every input. The URI's write permission is checked
before the attempt, so a grant that has been revoked from
Settings → App Info → Storage is cleared rather than retried on
every open; the file path is then read directly. When the URI read
is attempted and does not return `SUCCESS`, the file path is tried.
When neither source succeeds, `loadCurrentSongInViewMode()` keeps
the editor on the current song: the title, artist, album, and genre
fields are populated from the song database's row for the path, the
header stays on VIEW MODE, and MODIFY is enabled so the user can
re-establish file access through the storage access framework.
`onCreate()` wraps `Song.getContentUri()` and
`CharacterMapper.getSongFromFile()` in try/catch, so a malformed
cached URI or a file whose embedded tags cannot be read resolves to
a null field rather than to an exception. The editor always opens
on the song `MusicService.getCurrentSong()` names, whose playlist
entry is refreshed by `refreshMetadataForSong` when the
`METADATA_UPDATED` broadcast lands; a save, a close, and a re-open
therefore show the values the service now carries, not the previous
edit.

The editor treats the URI the current session selected as its write
target, and the absolute path the song database and the playback
consumers key on is always paired with that URI. When the editor is
opened with a `song_uri` extra, a supplied `song_path` is accepted
only when `getFilePathFromUri(uri)` resolves the URI to it, or when
`SongDatabase.getSong(suppliedPath)` returns a row whose `getUri()`
equals the supplied URI; otherwise `mFilePath` stays null. When the
editor is opened with only a `song_path`, the URI is read from the
database. The SAF picker result augments identity rather than
replacing it: the pick adds the URI, and a resolved path refines the
path but never removes one the caller already supplied. The URI is
authoritative for writing; the path is authoritative for matching
receivers. MODIFY prefers the session-selected URI over a stored URI
when the session has one with write permission, and persists it
against the current path through `saveUriToDatabase()` before
entering edit mode.

`performSaveChanges` captures the target URI, the target path, the
album-art-removed flag, the pre-edit bundle, and the submitted
update into `final` locals before it dispatches the write to the
executor, and passes them to the success and failure handlers as
parameters. A later SAF pick or a later change to `mGrantedFileUri`
or `mFilePath` cannot redirect a save that is already running.
`onSaveSuccess` builds the announcement from the submitted update
overlaid on the pre-edit bundle through
`buildAnnouncementFromSubmitted`, and the post-write reread is
verification only: a reread that returns `FILE_UNREADABLE` or
`PARSE_FAILED` logs a warning and the submitted values are
announced anyway, because the write itself succeeded. A reread
failure cannot suppress the broadcast, the database row, or
playback resumption. The shared album-art cache is evicted under
both the path key and the URI key before the broadcast. The
broadcast always carries the saved URI; the path is included when
the editor knows one. `onSaveFailure` takes the captured target
path as a parameter and, on `PERMISSION_DENIED`, writes the
fallback database row against that path instead of `mFilePath`.

The `METADATA_UPDATED` broadcast has three receivers. `MusicPlayer`
resolves the edited song's path through `resolveBroadcastSongPath`,
calls `MusicService.refreshMetadataForSong`, and invalidates the
carousel's per-path cache. `PlaylistActivity` resolves the same
path and updates the display row, refreshing the playing indicator
when the edited song is current. `MusicService` owns its own
receiver, registered in `onCreate` and unregistered in
`onDestroy`, so an edit lands on the in-memory playlist even when
the player activity has been reclaimed but playback continues. On
a SAF-only write, the editor knows the URI but not the path; each
receiver's `resolveBroadcastSongPath` helper resolves the URI
against `SongDatabase` so path-keyed consumers still find the
edited song.

**The overlay stack** lives in `OverlayManager` and `ToastManager`.
`OverlayManager` is the process-wide, reason-keyed dim-and-spinner
overlay. `ToastManager` is the operation-stack manager for persistent
overlays and temporary toasts. An operation is registered on the stack
by its ID; the stack renders the top operation and retains the lower
ones. Cancelling an operation is removing its entry from the stack,
which is indistinguishable from an operation that completed. A toast's
vertical position is chosen from the window's current height: the
manager holds two margins and picks the one that centres the toast in
the visible gap for the window the app is running in, so a toast is
centred in full-screen and in split-screen alike, in every activity,
without any activity having to announce its own layout state.

**The album-art carousel** lives in `AlbumArtCarousel`. It is a pure
renderer of the window the service publishes. A three-slot strip one
slot-width wide sits inside a viewport one slot wide; the strip's
middle slot is aligned to the viewport at rest. A horizontal drag
follows the finger, and on release the strip either commits to a
neighbour or settles back to rest. Every other transition — a
transport button, a headset key, an automatic advance, an incoming
file — is animated from the service's broadcast alone. The activity
forwards the window, the direction, and the restart the service named
to the carousel's `setWindow` method, and the carousel renders exactly
the animation they describe. A commit presets the confirmed window
before the slide starts, so the animation's end action adopts the
window synchronously and the strip never enters the snapshot-awaiting
state. A restart adopts the incoming window from the first frame and
runs the below-threshold nudge. Both legs of the nudge run on the
same spring constants — the outward leg carries the strip to the
displaced position and the return leg carries it back over the same
distance, on the same damping ratio and stiffness — so the two halves
of the gesture take the same time to settle and overshoot by the same
small amount. Every motion toward a rest position — the nudge's two
legs, a returned swipe, a refused press, a snapshot timeout — runs on
the same spring, so the strip's physical settle is the same wherever
the motion began. A non-neighbour change runs the cross-fade: the
outgoing art fades out on the emphasized accelerate curve and the
incoming art fades in on the emphasized decelerate curve with a
subtle scale-up, over the same total duration a commit takes. The
cross-fade waits for the incoming art to resolve before it begins, so
the outgoing art stays on screen while the incoming art is being
decoded and the transition the user sees is one cross-fade from the
outgoing art to the incoming art, never a placeholder in between.
Every transition is animated, and the animation is chosen from the
shape the service named: a change to a neighbour slides a full slot
width on the emphasized decelerate curve; a restart of the same song
with an advanced generation runs a below-threshold nudge driven by
spring physics with a small overshoot; a change to a path that is not
one of the previous window's neighbours runs the staggered
cross-fade, with an accelerate outgoing leg and a decelerate incoming
leg. Every animation ends in a single main-thread turn that renders
the carousel's new bitmaps and hands the same turn to the activity's
commit runnable, so the album art, the title, the artist, the seek
bar, the transport glyph, the notification, and the media session
land on one frame.

**The edge fast-scroll** lives in `EdgeScrollHelper`. It installs an
`OnItemTouchListener` on a `RecyclerView` that claims a vertical drag
once the finger has crossed the touch slop inside the list's leftmost
or rightmost 32dp. The drag then maps linearly onto the total
scrollable range: a full viewport-height of finger travel moves
through five percent of the range. The gesture engages only when the
content is at least three times taller than the viewport; on a shorter
list the edge zones defer to the RecyclerView's own scrolling. A tap
is never consumed. The playlist and the playback-scope dialog both
install it, so the edge width, engagement threshold, and scroll rate
are the same on both surfaces.

**Split-screen behaviour is per-activity.** Every activity declares
the same `configChanges` set and adjusts its own content when the
window resizes, instead of being recreated. `MusicPlayer` sizes,
shrinks, or hides the album art; when the album art hides, the
wrapper around it is hidden with it, the title-and-artist group is
pinned to the top of its weight slot at the album-art top's Y offset
so the title lands where the album art used to begin, and the
transport row's bottom padding is reduced so the group's slot has
room for the artist line. Every view below the group keeps its
position relative to the window bottom because the group's weight
slot absorbs the difference. `EqualizerActivity` toggles the band
wrapper's layout weight so the band controls take the space the
effect controls leave in the full layout, and take their natural
height in split mode — the band controls and the effect controls
share one scroll container, so in split mode the two sections scroll
together. `FolderManager` keeps the list visible in every window
size and shows its placeholder only when the list is empty.
`MetadataEditor` and `PlaylistActivity` let their existing scroll
containers absorb the overflow.

**The storage observer** lives in `StorageObserver`. It registers a
single content observer, for
`MediaStore.Audio.Media.EXTERNAL_CONTENT_URI`. A change notification
starts a quiet period; if no further change arrives inside the period,
the observer asks `PlaylistManager` to run a scan. Audio additions,
removals, and metadata edits — including the app's own edits, which
are flagged as self changes — trigger the scan. Image, video, and
download writes do not, because the audio MediaStore URI is the only
one observed.

**Localization is a cross-cutting concern, not a subsystem.** Every
user-visible string the app shows resolves from `res/values/strings.xml`
in the base locale and from a `values-XX/strings.xml` in each of the
fifteen additional locales. `LanguageManager` owns the selected
language tag, the supported-language set, and the `wrap(Context)`
helper that produces a context whose configuration carries the
selected locale. Every activity overrides `attachBaseContext(Context)`
and calls `wrap` before any state exists, so the framework reads from
the matching resource folder for the whole of the activity's lifetime.
A language change is a persisted tag, a `LANGUAGE_CHANGED` broadcast,
and a `recreate()` on each activity that receives it; no restart of
the process is needed. The supported set is declared three times —
`LanguageManager.LANGUAGE_TAGS`, `res/xml/locales_config.xml`, and
the `resourceConfigurations` block in `app/build.gradle` — and the
three lists describe the same set.

Every one of those strings is a resource, including the unit symbol
that appears on the equalizer's per-band frequency labels. Those
labels read `equalizer_frequency_hz`, `equalizer_frequency_khz`, and
`equalizer_frequency_khz_decimal`, and each locale supplies the unit
its language uses — `Hz`/`kHz` in the Latin-script locales, `Гц`/`кГц`
in Russian. The decimal separator between a kilohertz value and its
tenths is not in the resource: it comes from the JVM default locale,
which `LanguageManager.wrap` has already set to the app's selected
language, so the same string renders as `1.2 kHz` in English and
`1,2 kHz` in German.

The zero-value strings for semitones and decibels are parameterized
for the same reason. `equalizer_semitones_zero` is `%1$.1f st` and
`equalizer_db_zero` is `%1$.1f dB`, so the zero value is formatted by
the same code path that formats the positive and negative variants,
and the decimal separator follows the app's language. A translator
does not maintain a separate literal for zero.

The album-art cache is process-scoped. `AlbumArtCache` is a
singleton that outlives every activity, and both `MusicPlayer` and
`NotificationController` hold a reference to its underlying
`LruCache` rather than owning a cache of their own. A recreate does
not evict the bitmaps, and the carousel's first read after the
recreate finds the entry the previous instance decoded.

The top bar in every activity and every dialog lays its title out in
a horizontal linear layout with a weight of 1, so the title is the
only element that can yield. A translation longer than the space the
two edge controls leave truncates with an ellipsis rather than
running under a button beside it. The same reasoning is why the
separator between the sort-mode prefix and the mode label is a
Java-side constant rather than a trailing space on the prefix
resource: the resource compiler strips trailing whitespace from a
non-formatted string, and a translator cannot preserve a space they
cannot see.

Layout spacing uses explicit 8dp margins rather than string-based
separators in headers and list rows, ensuring a visible, consistent
gap across all screen sizes and languages. In right-to-left languages
like Arabic, the Music Player timers, the Credits version header, and
the dynamically generated Equalizer frequency bands use relative
`START` and `END` gravity to anchor perfectly to the outer 16dp edges
of their containers, ensuring a balanced and symmetrical UI in every
language. The Metadata Editor's form uses explicit `layout_marginTop`
values to guarantee uniform bounding boxes for every field label.

A language change closes the surface the choice was made from and
recreates the activity in one step. The dialogs do not rebuild
themselves in place. Every surface the user was on when the choice
was made is dismissed, and the activity is rebuilt with a wrapped
base context that carries the new locale. The next time a dialog is
opened it is constructed against the new locale from the start, so
every string and every direction-dependent layout attribute resolves
correctly from the first frame it renders.

The theme dialog's row and the preset dialog's row share one shape.
The row's own padding is symmetric and logical, so both its edges
carry the inset in every direction. Each child that sits inside
the row — the name, the colour preview circle, the `CURRENT`
indicator — carries at most a single logical start margin, which
describes the gap between it and its sibling. The trailing inset
on whichever child lands at the trailing edge is supplied by the
row's own end padding, not by a second margin on the child. That
gives the row one writer for each edge and one writer for each
gap, which is what keeps the layout correct under LTR and RTL
without any special-casing.

</details>

## Build from Source

### Prerequisites

- Android Studio Hedgehog or later
- JDK 17
- Android SDK API 35

### Build Commands

```bash
git clone https://github.com/oneadzido/llama-music-player.git
cd llama-music-player

./gradlew assembleDebug     # Debug build
./gradlew assembleRelease   # Release build
./gradlew clean             # Clean build
```

## First Time Setup

1. Tap the MANAGER button on the main screen
2. Tap IMPORT FOLDER to select your music folder
3. Tap UPDATE to save your selection
4. The app will scan your folder and build the playlist

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

## Privacy

Llama Music Player does not collect, transmit, or store any personal information. See [PRIVACY.md](PRIVACY.md) for the full policy.

## Contact

**Richard Korbla Adzido**
oneadzido@gmail.com