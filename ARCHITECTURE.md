# Architecture

This document describes how Llama Music Player is organized, why
certain decisions were made the way they were, and where the seams
are. It is written for someone reading the source for the first time,
and for the author's own future self.

The app is arranged as the systems of a human body. That is not a
decoration. It is how the class structure was decided: when a new
system was needed, the question asked was *what part of the body would
do this job?*, and the answer decided which class would own the fact
and which class would be allowed to read it. The rest of this document
uses the same scheme, because using a different scheme would describe
a codebase the author did not build.

---

## The four principles

Every rule in the codebase is a consequence of one of these four. When
a new case arrives that the existing patterns do not cover, these are
what decides it.

**One writer per fact.** Every piece of state has exactly one method
that writes it. Every other method reads it. When a reader asks
"where does this value come from?", the answer is one method, not a
set of methods that could disagree. The media session has one writer,
`NotificationController`. The current language has one writer,
`LanguageManager.setLanguage`. The decoded bitmaps have one owner,
`AlbumArtCache`. The favorite column has one writer on an existing
row, `SongDatabase.setFavorite(String, boolean)`. The active playback
scope has one writer, `MusicService.setPlaybackScope`. The audio focus
flags (`mHasAudioFocus`, `mIsDucked`) have one writer,
`MusicService.handleAudioFocusChange(int)`.

**One shape per role.** A class that plays a role — singleton, state
machine, ticker, safely-suffixed helper, state listener — has the
same shape it has in every other class. The role determines the shape,
not the class. Every singleton with no construction-time side effects
uses the same double-checked locking pattern. Every safely-suffixed
helper takes a nullable parameter, returns early on null, catches
`Exception`, and writes the reason in a comment.

**One name per concept.** The same concept carries the same name in
every file. `mMainHandler` is the handler bound to the main looper.
`getInstance` is the singleton accessor. `*Safely` is the "swallow
the exception, the failure is not interesting" suffix. There is no
second name for any of these.

**Document the why, not the what.** KDoc explains the reason a
method exists, the invariants it holds, and the failure modes it
guards against. The method body shows the what; the KDoc shows
the why.

---

## The signalling contract

Every signal that used to travel between components as an Android
broadcast now travels on the process EventBus. The façade is
`com.ark.llama.event.LlamaEventBus`, installed once at the top of
`LlamaMusicPlayer.onCreate`, and the set of events is grouped by
producer surface in the `com.ark.llama.event` package:

- `MetadataEvent` — `Updated`, `GenreUpdated`.
- `PlayerEvent` — `Snapshot` (sticky), `ApplyScope`.
- `PlaylistEvent` — `LibraryReset` (sticky), `Refresh`, `Display`,
  `SyncStarted`, `SyncComplete`, `TriggerGenreLoading`, `SongAdded`.
- `SessionEvent` — `AudioSessionChanged`, `PlaybackParamsRefresh`.
- `SettingsEvent` — `LanguageChanged` (sticky), `ThemeColorChanged`
  (sticky).
- `TransportEvent` — nine action-shaped carriers
  (`TogglePlayPause`, `Next`, `Previous`, `Resume`, `Pause`, `Stop`,
  `ToggleFavorite`, `RequestResumeFromGesture`, `SavePlayerState`)
  and eight data-shaped carriers (`PlaySong`, `PlaySongByIndex`,
  `PlaySongAtPosition`, `SeekTo`, `SetPendingSong`, `SetRepeatMode`,
  `SetShuffle`, `SetBluetoothConnected`, `SetPlaybackSortMode`).

`LlamaEventBus.init()` is `@Synchronized` and idempotent, and
installs the default bus with the library's reflective subscriber
lookup. It is called once, at the top of `LlamaMusicPlayer.onCreate`,
before any manager whose constructor registers a subscriber. The one
`@PublishedApi internal` constant is `TAG`, referenced by the two
public inline functions `latest<T>()` and `removeLatest<T>()`; the
Kotlin compiler rejects a public inline function that reads a
non-public member, and `@PublishedApi` opens exactly that one
reference without widening the constant's visibility.

The one exception to "no broadcasts" is the system
`VOLUME_CHANGED_ACTION` broadcast that `MusicService.mMusicVolumeReceiver`
consumes. It is emitted by the framework when the user presses a
hardware volume key; the app is a consumer of the system, not a
producer of an internal event. Its receiver stays registered in
`onBind` and unregistered in `onDestroy`, and no other
`BroadcastReceiver` in the app consumes a `sendBroadcast` from
another app component.

The notification transport actions the `NotificationController`
attaches to the notification buttons (`ACTION_PLAY_PAUSE`,
`ACTION_NEXT`, `ACTION_PREV`, `ACTION_STOP`) are `PendingIntent`
actions the system delivers to `MusicService.onStartCommand`. They
are not app-internal broadcasts and they are retained. The
`BluetoothReceiver`'s `startForegroundService(ACTION_BLUETOOTH_RESUME)`
path is the cold-process entry into `MusicService`; it is not an
internal signal either.

---

## The systems of the body

### The heart

`MusicService`, `PlaybackWindow`, `AlbumArtCarousel`,
`PlaybackScope`, `PlaybackScopeDialog`.

The heart is `MusicService`, and its job is to pump audio through the
app. A single `ServiceState` enum, two monotonic counters, and a
two-index model (`mCurrentIndex` and `mTargetIndex`) hold every
transition-related value. State mutations run on the main thread; raw
`MediaPlayer` calls run on a dedicated `HandlerThread`. Only
marshalled `Runnable`s cross between them.

The heart has one method that expresses the display-song rule. It
is not a two-line accessor; it is a guarded evaluation of "which song
should the carousel be sliding toward" that returns null when the
answer is not yet available.

```kotlin
private fun getDisplaySong(): Song? {
    val list = mCurrentPlaylist ?: return null
    if (list.isEmpty()) return null
    if (mState == ServiceState.PREPARING) {
        if (mTargetIndex < 0 || mTargetIndex >= list.size) return null
        val target = list[mTargetIndex]
        if (target == null || target.path.isEmpty()) return null
        return target
    }
    if (mCurrentIndex >= 0 && mCurrentIndex < list.size) {
        val current = list[mCurrentIndex]
        if (current == null || current.path.isEmpty()) return null
        return current
    }
    return null
}
```

During a transition, the display song is the target. Otherwise it is
the current song. The carousel, the player UI, the art prefetch, the
notification, and the media session all read this method directly.
During a transition the display song is the target, so all five
surfaces begin advancing toward the incoming song on the same
main-thread turn a transition begins, and all five land on the
incoming song on the same frame the transition commits.
`snapshotForNotification()` calls `getDisplaySong()` unconditionally,
with no branch on state, so the notification renders exactly the
song the player screen renders for the same generation. There is one
expression of the display-song rule and one answer to "which song is
the app showing" for every consumer.

The guards are the method's contract. The playlist can be empty
during a library reset. `mTargetIndex` can point past the end of
the playlist during the window between a `setPlaylist` replacement
and the next reconcile. The song at the index can be null during
a database read that has not yet completed. The display song's
path can be empty during a loose-track insert that has not yet
written the row. Every one of those cases returns null.

The consumers of the method treat null as an instruction to hold
their current surface, rather than to clear it.
`snapshotForNotification()` returns null, and both
`commitFromBroadcast` and `syncOnce` refuse to render that null,
leaving the notification on its last rendered snapshot. The carousel
holds its last window. The player UI holds its last drawn title and
artist. The surface clears only when the state machine itself
transitions to `IDLE`, which the controller checks through
`ServiceStateSource.isIdle()`.

The consequence of the rule is visible in the app. Tap Next, and the
carousel begins sliding toward the incoming song's art and the
notification begins describing the incoming song on the same frame.
When the prepare completes and the transition commits, every surface
advances to the incoming song together, on the same frame.

The blood the heart pumps is `PlaybackWindow`: an immutable value
holding three paths — the previous neighbour, the current song, the
next neighbour — plus the service's monotonic transition counter. The
service computes it; the carousel consumes it. The carousel holds no
playlist, no index, no bitmap as primary state.

The heart also owns the three-tier playlist. `mFullPlaylist` is the
immutable database reflection, `mScopedPlaylist` is the same list
filtered by the active `PlaybackScope`, and `mCurrentPlaylist` is the
active queue — the scoped playlist plus the grace-period songs at
`mCurrentIndex` and `mTargetIndex`. `rebuildPlaylists()` is the single
mutator for the second and third tiers. The favorite column is
written only by `SongDatabase.setFavorite(String, boolean)` and by
the fresh-row branch of the insert; every other metadata write goes
through `SongDatabase.updateSongMetadata(Song)`, which deliberately
omits that column, so a scan cannot clobber a user's favorites.

The scope itself is `PlaybackScope`: an immutable value with seven
kinds — All Songs, Favorites, Loose Tracks, Specific Folders,
Specific Genres, Specific Artists, Specific Albums — and four filter
sets, of which exactly one is populated for the four filter kinds.
`matches(Song, Set)` is the only read; it consults
`CharacterMapper.isUnknownX(String)` so a song whose field is the
current locale's placeholder is never matched against a stored
value. A malformed serialized scope deserializes to
`PlaybackScope.ALL_SONGS`, so a corrupted preference cannot leave
the player with a scope that matches nothing. `MusicService` persists
the active scope under `playback_scope_json` and exposes it through
`getPlaybackScope()` and `setPlaybackScope(PlaybackScope)`; the latter
is the single writer of the fact. `PlaybackScope` is a Kotlin class
whose `paths`, `genres`, `artists`, and `albums` fields are Kotlin
`Set<String>` built with `.toSet()`, so the field type and the
serialized JSON agree on which interface is being carried across
the serialization boundary.

`PlaybackScopeDialog` is the UI through which the scope is chosen.
It presents the seven modes as a single flat list and expands the
selected mode's filter list below its row. The header reads
`SCOPE MODE: <label>` and carries a live count of the songs the
current selection would yield, computed through the same
`PlaybackScope.matches(Song, Set)` matcher the service uses to
build the playlist. Because the dialog and the service share one
matcher, the number the dialog shows and the queue the service plays
cannot disagree. The dialog does not bind `MusicService`; it returns
the selected scope through `OnScopeAppliedListener`, and
`FolderManager` posts a `PlayerEvent.ApplyScope` with the serialized
scope on the bus. The service is the single writer of the active
scope.

Song completion is evaluated against `mScopedPlaylist`, not the
grace-period-extended `mCurrentPlaylist`. The grace-period entries keep
the current and pending songs alive while a scope changes; they do not
become candidates for the next completed-song selection. Repeat off
advances within the scope and pauses when the scope is empty or the
current song is its final entry. Repeat all wraps within the scope,
shuffle selects from it, and repeat one continues the current song.

The heart also owns the audio-focus record. A loss that interrupts a
live transition is written once, as an `Interruption`, and a
subsequent loss of a different kind does not overwrite it, so the
dialer's `LOSS`-then-`TRANSIENT` sequence produces one record and the
gain that follows resumes from it. A successful resume clears the
record, and a later `AUDIOFOCUS_LOSS` replaces a stale
`TRANSIENT_LOSS` record with a `PERMANENT_LOSS` one. The record is
cleared by every user-initiated transition and by a transition to
idle. It drives three decisions: whether a gain resumes playback,
whether the notification stays foreground through the interruption,
and whether a subsequent loss of a different kind is ignored.

The volume-key rule is part of the same record's contract. A hardware
volume press while the player is backgrounded does not take audio
focus from an app that is playing. The volume-raise path in
`onMusicVolumeChanged`, the Bluetooth volume remap in
`handleMediaButton`, and the media-session `onPlay` callback all
require the service to already hold focus; an interruption record
without focus names an interruption the user has not overridden,
and the app does not act on it. The eventual `AUDIOFOCUS_GAIN`
resumes through `resumeAfterInterruption()` — the same entry point
the gain callback uses — so a volume press and a focus return take
the same path once the interrupting app has released the stream.

The audio-focus request the service issues for a resume is
`AUDIOFOCUS_GAIN_TRANSIENT`. A resume of a paused track is a bounded
interruption of whatever was playing; the permanent request the
service uses to acquire focus on first playback is unchanged. The
transient request is less likely to be honoured against a competing
permanent holder, which closes the remaining device-dependent path
to a steal.

The design pays off in a fact that is easy to miss. When a song
restarts under repeat-one, the three paths the window carries are
unchanged. What has moved is the generation, and what the service has
set alongside it is a restart flag. The carousel sees the fresh
generation, reads the flag, and knows that the song it was already
showing has begun again rather than that a different song has taken
its place. It runs a small nudge out to the neighbour and returns.

The carousel does not know what repeat-one is. It reads a number and a
boolean, and responds to what they say.

`AlbumArtCarousel` is the pulse that shows the transition on the
album art. It has seven states — `IDLE`, `DRAGGING`, `COMMITTING`,
`AWAITING_SNAPSHOT`, `RESTARTING`, `REVEAL_PENDING`, `REVEALING` —
and each one describes a distinct phase of a transition. Nothing is
left over. Nothing is missing.

`MusicService` owns the audio session for its full service lifetime.
Its `MediaPlayer` is created, operated, and released on the dedicated
player thread; the service's main thread owns state transitions and
notification commits. Foreground promotion follows playback state, not
the visibility of `MusicPlayer`, so an activity recreation does not
interrupt a session that is still playing.

The service's collection declarations use Kotlin's `Set` and `Map`,
not `java.util.Set` and `java.util.Map`. `mLooseTrackPaths` is a
Kotlin `Set<String>`; `commitFavorites(changes: Map<String, Boolean>?)`
accepts a Kotlin `Map` and iterates it with
`for ((path, favorite) in changes)`. This is what allows
`PlaylistActivity`'s `HashMap<String, Boolean>` to be passed without
a cast and what allows the destructuring `for` loop over `changes` to
compile. The `LruCache` for the notification art is built with an
`object : LruCache<String, Bitmap>(size)` expression rather than a
trailing lambda; the trailing-lambda form is not valid for the
constructor overload the library exposes, and `object :` is the
Kotlin spelling that wires `sizeOf` correctly. The session-token
provider is installed with `{ getSessionToken() }`, a call to the
public accessor, not with `{ sessionToken }`, which would be a
property reference the class does not expose.

### The lungs

`PlaylistManager`.

The lungs breathe life into the playlist. Folder scans, genre loads,
and library resets are all `PlaylistManager`'s job, and each of the
three is an operation on the `NavigationController` stack that
declares its own liveness window. The reset operation has a write
barrier, an invalidation event, and a completion signal, and its
structure follows four rules: one owner, one write barrier, one
invalidation signal, one completion signal. Every signal it produces
travels on the bus: `PlaylistEvent.LibraryReset`,
`PlaylistEvent.SyncStarted`, `PlaylistEvent.SyncComplete`,
`PlaylistEvent.Refresh`, `PlaylistEvent.Display`,
`PlaylistEvent.TriggerGenreLoading`, and
`MetadataEvent.GenreUpdated`.

### The nervous system

`NavigationController`, `MusicPlayer`.

`NavigationController` is the nerves that coordinate actions across
the app. It enforces *declared liveness*. Each operation announces how
often it will heartbeat and how many missed heartbeats it tolerates.
The controller does not guess. It reads the declaration and enforces
it. An operation that heartbeats every 500 milliseconds and tolerates
three misses has a fifteen-hundred-millisecond window, and the
controller does not need to know why the window is that size. Its
`State` enum is public — not `internal` — because the public
`StateListener.onStateChanged(oldState, newState)` method exposes it
in its signature, and Kotlin rejects a public member that references
a type narrower than the member itself.

`MusicPlayer` is the brain — the main surface, coordinating everything
the user sees. It is the largest class in the codebase, and the next
section explains why that is a decision rather than an accident.

The brain also owns the album-art gesture classifier. A single touch
on the album-art container is classified on its first significant
move into `TOUCHED`, `CAROUSEL`, `VOLUME`, `PLAYBACK`, or `CANCELLED`.
The four gestures the album art supports — the carousel swipe, the
volume drag, the centre swipe to play or pause, and the double-tap
favorite — share one state enum so a touch cannot fall into two
modes at once. The `PLAYBACK` gesture requires twice the touch slop
of travel and commits on release; the `TOUCHED` state is the
resting state from which the double-tap favorite is detected and
dispatched through `handleGestureDoubleTap()`.

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

### Memory

`MusicLibrary`, `SongDatabase`, `AlbumArtCache`, `Song`.

Memory is split by lifetime. `MusicLibrary` is the working memory —
the songs loaded for this session. `SongDatabase` is the long-term
memory, holding two tables with identical schemas: `songs` for
imported folders and `loose_songs` for tracks opened from other apps.
Reads resolve column indexes once per query and reuse them across
rows, because the six column lookups per row over a multi-thousand-row
result dominate the cost of a full table read. `AlbumArtCache` is the
visual memory — the decoded bitmaps, held process-wide because a
bitmap is a fact about a song, not a fact about a screen. `Song` is
a single memory: one song and its facts. Each activity or service
owns the `MusicLibrary` instance it creates. When that component is
destroyed, the library stops accepting work without waiting on the
main thread; queued completion and progress callbacks check the
owner's lifetime before reaching the UI.

`SongDatabase` is a Kotlin class whose methods are
`fun getAllSongs()` and `fun getLooseSongs()`, not synthesized
properties. Kotlin does not synthesize property access for Kotlin
functions — only for Java getters — so every caller writes
`getAllSongs()` and `getLooseSongs()` explicitly. `getExistingPaths`
returns `Set<String>` built from a Kotlin `HashSet`;
`deleteSongsNotInSet` returns `Int` and takes a `Set<String>?`;
every one of these is a Kotlin type, not a `java.util` type.
`SongDatabase` also has a `setFavorite(String, Boolean)` method that
is the only writer of the favorite column on an existing row, and
an `updateSongMetadata(Song)` method that is deliberately blind to
that column.

`MusicLibrary` has a `val songCount: Int get() = getSongCount()`
property so a caller that wants the count without a defensive copy
can read `library.songCount` rather than paying the copy cost of
`getAllSongs().size`. The `songCount` property reads
`getSongCount()`, which locks the read lock and returns the size of
the in-memory list. `getAllSongs()` locks the read lock and returns
`ArrayList(mAllSongs)` — a copy — so the caller cannot mutate the
library's state through the returned list. The two are the same
number read through two different methods; the difference is the
copy.

### The senses

`EqualizerManager`, `StorageObserver`, `CharacterMapper`.

`EqualizerManager` is the ears. It owns the audio effect objects and
the persisted settings. SharedPreferences are the authoritative store;
getters read what setters wrote, regardless of any rounding the
underlying effect applies. Its calls into
`android.media.audiofx.Equalizer` pass a `Short` band index
(`band.toShort()`), because that is what the framework's JNI surface
expects and Kotlin will not widen `Int` to `Short` implicitly.
`getCenterFreq(band)` is `getCenterFreq(band.toShort())`;
`setBandLevel(band, level)` is
`setBandLevel(band.toShort(), level.toShort())`;
`getBandLevel(band)` is `getBandLevel(band.toShort())`. Every call
site in `attachToAudioSession`, `setBandLevel`, `getBandLevel`,
`applyCustomPreset`, `applyStandardPreset`, and
`clearCustomBandLevels` follows that rule. The band label above each
slider reads its unit symbol from a resource, so a Russian device
shows `Гц` and every other device shows `Hz`. Its audio-session and
playback-params signals reach the service and any other consumer
through `SessionEvent.AudioSessionChanged` and
`SessionEvent.PlaybackParamsRefresh`.

`StorageObserver` is the listening. It registers a single content
observer for the audio MediaStore URI. Audio additions, removals, and
metadata edits trigger a quiet period and then a scan; the observer
posts `PlaylistEvent.Refresh` and
`PlaylistEvent.TriggerGenreLoading` on the bus. Image, video, and
download writes do not.

`CharacterMapper` is the reading. When a tag is absent, unreadable, or
carries a recognized placeholder value, the parser produces a
canonical value in the app's current language. The value is resolved
from a string resource at the moment it is needed, so a German library
shows the German placeholder for a song whose artist tag is missing,
and an Arabic library shows the Arabic one. Its
recognized-placeholder set is a Kotlin `Set<String>`, initialized to
`emptySet()` and built from
`LanguageManager.getInstance(context).getLanguageTags()`; the
localized context is read through
`LanguageManager.getInstance(context).getLocalizedContext()`, not
through a property.

### The reflex arc

`BluetoothReceiver`.

A reflex arc is a small circuit — a sense, a nerve, a muscle, no
brain in the loop. `BluetoothReceiver` is exactly that. When a headset
or A2DP profile connects or disconnects, the receiver posts
`TransportEvent.SetBluetoothConnected(true)` or
`TransportEvent.Pause` on the bus. `MusicService.onTransportPause`
consumes the pause. The service owns the pause because the service
owns the audio, and the pause fires whether or not the player
activity is alive. The receiver also posts
`TransportEvent.SetBluetoothConnected(false)` on disconnect, which
the service consumes to update its Bluetooth state. The receiver
imports `com.ark.llama.event.LlamaEventBus`, not
`com.ark.llama.event.Bus`, and calls `LlamaEventBus.post(...)`, not
`Bus.post(...)`.

### The hands

`MetadataEditor`, `MetadataReader`, `MetadataWriter`,
`StorageAccessHelper`.

The hands are what modify the music files. `MetadataEditor` is the
activity that shows the two hands — VIEW MODE reads, EDIT MODE
writes. `MetadataReader` is the reading hand, pulling tags out of a
file. `MetadataWriter` is the writing hand, putting tags back without
corrupting the file: copy, backup, edit, verify, copy back. If any
step fails after the backup, the original is restored. The result of
the write is an eight-value status enum, and one method on it answers
the only question a caller should have to ask. `StorageAccessHelper`
is the grip — holding the permissions that let the hands reach the
files. Its `getAllFolderUris()` returns a Kotlin `Set<String>`
built with `.toSet()` from the internal HashSet; its
`getDocumentUriForFile` uses `current?.findFile(part)` and assigns
`current = child` after the null check, so the nullable
`DocumentFile?` chain resolves without an assertion.

The editor publishes a single `MetadataEvent.Updated` event after a
successful write, via `LlamaEventBus.post`. The player, the playlist
activity, and the service each subscribe to that event. There is no
`METADATA_UPDATED` broadcast; the notification receives the edit
through the same `PlayerEvent.Snapshot` that the service posts when
it refreshes its in-memory playlist from the edit.

`MetadataReader` declares its writable-extension set as a Kotlin
`setOf(...)`, not as a `HashSet(Arrays.asList(...))`. The Kotlin
`setOf` returns a `Set<String>` that matches the declared field type;
the Java-style construction returned a `java.util.HashSet` and
required a cast.

### The voice

`ToastManager`, `NotificationController`.

The voice speaks in two registers. `ToastManager` is the voice inside
the app, holding an operation stack. An operation is pushed with an
identifier, and the stack renders the top operation's message. A
lower operation is retained but not shown. Cancelling an operation is
removing its entry, which is indistinguishable from an operation that
completed. The stack does not pretend to know what it cannot know;
callers that need to distinguish those two cases carry their own
terminal state. The overlay's text view is created with
`setHorizontallyScrolling(false)`, not with the nonexistent
`isHorizontalScrollingEnabled` property.

`NotificationController` is the outward voice — speaking from the
notification and the lock screen. It is the sole writer of the media
session's metadata *and* its playback state. Those are two different
projections of the service. The metadata changes when a song is
committed. The playback state changes on every tick of the state
machine. The two projections have independent lifetimes, they are
written from the same source in the same main-thread turn. The
notification render is committed only after its render succeeds; an
unsuccessful generation remains eligible for the next synchronization
pass. If a render fails after the media session has been updated, the
controller reapplies the previous snapshot so both surfaces continue
to describe the same committed state. Posting the notification and
promoting the service to the foreground are separate operations, and
a sticky service restart reconstructs a paused notification from the
persisted playback state. While the service is reconciling a prepare
failure or player error, its pending-transition state prevents the
notification's cancel path from removing the visible playback surface.

The controller reads its rendered content from the service through
`ServiceStateSource.snapshotForNotification()`, which calls
`getDisplaySong()` unconditionally. The notification therefore shows
the target during a transition, the same song the carousel and the
player screen are showing, and advances with them on the same frame.
The `PlayerEvent.Snapshot` that reaches the activity and the
`commitFromBroadcast` call that reaches the controller carry the
same generation and the same payload.

The notification's own transport buttons dispatch directly to
`MusicService.onStartCommand` through `PendingIntent.getService`
with the private action `ACTION_PLAY_PAUSE`, `ACTION_NEXT`, or
`ACTION_PREV`. `MusicService.onStartCommand` handles each
synchronously on the main thread by calling `togglePlayPause()`,
`playNextSong()`, or `playPreviousSong()`. This is one hop from tap
to state-machine method. The media session is not in the path. The
media session exists to receive input from sources the app cannot
intercept — the lock screen, a headset, a Bluetooth button — and
those still go through `ACTION_MEDIA_BUTTON` and
`MediaSessionCompat.dispatchMediaButtonEvent`. The notification's
buttons are app-internal, so they take the direct path. Routing
them through the media session would add two hops through the main
queue, during which a player-screen tap that arrived after the
notification tap could execute first, and the two surfaces would
end up in disagreement with the user's last tap.

The controller imports `com.ark.llama.event.LlamaEventBus`. Its
`@Subscribe onThemeColorChanged` is registered on the sticky
`SettingsEvent.ThemeColorChanged` — its body is a no-op because the
notification chrome is rebuilt from the persisted theme on the next
commit.

### The skin

`ThemeManager`, `ThemeHelper`.

`ThemeManager` is the skin's color. `ThemeHelper` is the skin's touch,
painting every control. One value drives every button, every seek bar,
every cursor, every tinted placeholder, and the notification's own
large icon. The theme's *name* is a resource, so a German theme list
says "Königliches Lama" and an Arabic one says "لاما ملكي", and both
refer to the same color value in the same array.

A theme change is a rebuild, not a repaint.
`ThemeManager.setThemeColor(int)` persists the value and posts a
sticky `SettingsEvent.ThemeColorChanged` on the process bus via
`LlamaEventBus.postSticky`; every activity subscribes in STICKY mode
and answers with a posted recreate, every open dialog dismisses
itself, and every surface is re-constructed against the persisted
color. `ThemeHelper` still paints each newly-constructed view; it no
longer has a live view tree to repaint on an event, and the theme's
color is read at construction rather than patched in. The same shape
is used for `SettingsEvent.LanguageChanged`: the sticky event is a
signal to rebuild, not an instruction to patch.

`ThemeHelper.themeColorInt` is a property, not a method. Every caller
that wants the theme as an `Int` reads `themeHelper.themeColorInt`;
the name `getThemeColorInt()` does not exist. `ThemeHelper` also
declares `updateButton`, `updateButtonPendingState`,
`updateControlButton`, `updateSeekBar`, `updateEditText`,
`updateEditTextCursor`, `updateTextView`, `updateRadio`, and
`updateCheckbox`. `updateControlButton(button: TextView?, ...)` and
`updateButtonPendingState(button: TextView, ...)` differ in the
nullability of their first parameter; a caller holding a
`Button?` must pass it through `?.let { ... }` to the non-null
overload and can pass it directly to the nullable one.

### The organs the user touches

`PlaylistActivity`, `EqualizerActivity`, `FolderManager`.

Three activities are the surfaces a user spends most of their time in.
`PlaylistActivity` is the playlist — ten sort modes, a live search, an
edge fast-scroll, and a per-row heart that stages favorite changes in
`PendingFavorites` and commits them in one batch on `onPause`. The
row's own click handler shows the favorite confirmation toast from
the adapter's turn; the activity stages the flag change and commits
the batch. The header reads `SORT MODE [TITLE A-Z]` or
`SORTIEREN [TITEL A-Z]` or `СОРТИРОВКА [НАЗВАНИЕ А-Я]`, because the
label after the bracket comes from a resource and the space between
the two halves of the phrase is supplied by a Java-side constant.
`mCurrentDisplayList` is a Kotlin `ArrayList<Song>`, which
implements Kotlin's `List<Song>`; passing it to
`PlaylistAdapter.updateList(newSongs: List<Song>)` requires no cast.
`PlaylistAdapter.FavoriteListener` is a regular Kotlin interface,
not a `fun interface`, so `setFavoriteListener` is called with
`object : PlaylistAdapter.FavoriteListener { override fun
onFavoriteToggle(songPath: String, favorite: Boolean) { ... } }`
and not with a lambda.

`EqualizerActivity` is the equalizer — ten bands and sixty presets,
with every seek bar snapping to a step under a guard flag so a
programmatic re-set does not re-enter the listener that performed the
snap. The reset button reports *no change* when nothing has moved.
`MusicService.isPlaying` is read as a property (no parentheses);
`MusicService.isPausedOrReady()` is read as a method (parentheses);
the two are different kinds of member on the same class and the call
sites respect that. The equalizer's `themeColorInt` reads are
`themeHelper.themeColorInt`, not `themeHelper.getThemeColorInt()`.
The `serviceConnection` field is `ServiceConnection?`, and
`unbindServiceSafely` calls `serviceConnection?.let {
unbindService(it) }` so the nullable type does not force an
assertion.

`FolderManager` is the folder organ — speaking Storage Access
Framework, taking a persistable permission, mapping the URI to a
filesystem path, and cleaning up the loose tracks when a folder is
removed. Its header carries the SCOPE button, which opens the
playback-scope dialog; the dialog returns the serialized scope
through its `OnScopeAppliedListener`, which is implemented with
`object :` at the call site because it is a regular Kotlin interface
rather than a `fun interface`. `FolderManager` then posts
`PlayerEvent.ApplyScope` on the bus, so `FolderManager` itself does
not bind `MusicService`.

### The face

`CreditsDialog`, `LanguageDialog`, `PresetDialog`, `ThemeDialog`.

The face is what tells the user what the app is. `CreditsDialog` is
the front of it. Its version header carries the VERSION label, the
language button, and the version number. `LanguageDialog` is the
speaking face — picking the language. `PresetDialog` is the
equalizer's face. `ThemeDialog` is the skin's face.

Four dialogs share one top-bar shape: the Cancel button at the start
with its natural width, the title in the middle with a weight of 1,
and a trailing placeholder that reserves no width. The title is the
only element that can yield, so a long translation truncates with an
ellipsis rather than running under a button beside it. The corner
radii on every dialog are constructed with `.toFloat()` on each
element of the `floatArrayOf(...)` array, because `dpToPx` returns
`Int` and `GradientDrawable.cornerRadii` expects `FloatArray`.

A theme change made from `CreditsDialog`'s picker closes every dialog
on the screen — the picker, the `ThemeDialog` it opened, and the
`CreditsDialog` underneath — on the same sticky event the hosting
activity recreates on. There is no window in which one dialog has
been rebuilt and another has not, because no dialog is rebuilt in
place; all of them are dismissed, and the next open constructs a new
dialog against the persisted theme from the first frame.

### The fingers

`PlaylistAdapter`, `FolderAdapter`.

The fingers touch the rows. Each adapter renders one list. Neither
holds the data; the activity owns it, and the adapter asks for what it
needs on each bind. `PlaylistAdapter`'s subtitle joins artist, album,
and genre with a bullet, and each field is checked against the
corresponding `CharacterMapper` predicate before it is rendered, so a
field that carries a placeholder from any supported locale renders the
current locale's placeholder in its place. The adapter's trailing heart
icon flips the row's `Song` favorite flag, forwards the change to a
`FavoriteListener`, and — when the adapter's context is an
`Activity` — shows the favorite confirmation toast on the same turn.
The owning activity stages the flag change in `PendingFavorites` and
commits the batch on pause. The toast is the adapter's event and the
adapter's turn; the persistence is the activity's. The two owners do
not overlap.

`PlaylistAdapter` declares `updateList(newSongs: List<Song>)` with
Kotlin's `List` — the same interface `ArrayList<Song>` implements —
so passing the activity's `ArrayList<Song>` requires no cast. The
adapter's `onBindViewHolder` overload that takes a payload list uses
`MutableList<Any>`; this is the signature the library's own Kotlin
mapping produces from `List<Object> payloads`.

### The immune system

`PermissionChecker`.

The immune system decides what the app is allowed to touch. The set of
runtime permissions the app requests is version-gated — the media-read
permission the app asks for is `READ_MEDIA_AUDIO` on API 33 and later
and `READ_EXTERNAL_STORAGE` below that, `POST_NOTIFICATIONS` is
required from API 33, the foreground-service media-playback permission
is required from API 34, and the Bluetooth connect permission is
required from API 31. The set lives here, in one place, so every
reader of "are the permissions granted" reads the same answer. Four
readers use it: the player's initialization flow, the player's resume
check, the transport controls' enabled state, and the folder
manager's mutation gate. Because the four readers share this class,
they cannot disagree about what "permitted" means.

### The skeleton

`FontAwesome`, `LanguageManager`.

The skeleton is the structure that everything else is built on.
`FontAwesome` is the alphabet of shapes — the icons the app writes
with. `LanguageManager` is the tongue. Every user-visible string the
app shows is a resource, and the set of supported languages is
declared three times:

- `LanguageManager.LANGUAGE_TAGS` — the app's own list
- `res/xml/locales_config.xml` — the platform's per-app language picker
- `app/build.gradle.kts`'s `resourceConfigurations` — the release
  artifact's bound set

Three lists, one set. The set could have been derived at runtime from
the resource folders, which would have failed the moment a locale file
was added without its tag being registered. The set could have been a
single list in the app's own code, which would have left the platform's
picker and the release artifact unaware. Three lists, kept in sync by
hand, because they have three consumers with three different needs.

### The eyelid

`OverlayManager`.

The eyelid dims the view while work is happening. It is process-wide
and reason-keyed: four reasons are in use — a cold-start library load,
a metadata read, the folder-update cycle, a playlist sort — and each
caller holds its own. The class's invariants are enumerated one
through six, and each is a promise the class makes to itself: the
overlay is attached if and only if the reason set is non-empty;
`onActivityDestroyed` never touches the reason set.

### The edge fast-scroll

`EdgeScrollHelper`.

The fingers that scroll a long list have a second sense. A vertical
drag that starts inside the list's leftmost or rightmost 32dp scrolls
the list proportionally: a full viewport-height of finger travel maps
to five percent of the total scrollable range, anchored at the
position the drag began. The gesture engages only when the content is
at least three times taller than the viewport; a tap is never
consumed, so tapping a row near an edge still fires the row's click.

The helper is a standalone class, not a base activity. It installs an
`OnItemTouchListener` on any `RecyclerView` through a single
`attach(RecyclerView)` call. `PlaylistActivity` and
`PlaybackScopeDialog` both call it, so the edge width, engagement
threshold, and scroll rate are the same on both surfaces by
construction. A change to any of the three takes effect on both at
once, because there is one implementation to change.

---

## Cross-cutting concerns

### Threading

Three rules:

1. State mutations run on the main thread.
2. Raw `MediaPlayer` calls run on a dedicated `HandlerThread`.
3. Bitmap decoding and playlist reorders run on dedicated single-thread
   executors.

Only marshalled `Runnable`s cross between the main thread and the
player thread. Only the results of executor work — never the
intermediate state — are posted back to the main thread.

### The commit turn

Every transition the user can cause ends in a single main-thread
turn. The album-art carousel's new bitmaps are installed, the title
and artist are written, the seek bar's range is set, the transport
glyph is updated, the notification is committed, and the media
session's metadata is written, all on the same frame. Because view
mutations are batched by the framework into the next layout pass, the
art does not land before the text; both land together.

A resume is a commit turn too, and it commits synchronously. The
state machine enters PLAYING before the async `player.start()` is
dispatched to the player thread, so every surface flips on the same
main-thread turn the user's tap arrives, not on the player thread's
schedule. The async continuation exists only to roll the state back
if `player.start()` fails. The rollback is guarded by both
`mStartToken` and `mState`, so a pause that follows a resume within
one main-thread turn is not clobbered by a start that began before
it. `doPause` bumps `mStartToken` at entry precisely so that a pause
arriving mid-resume invalidates the resume's continuation.

`broadcastUpdate` is the only path that commits the notification.
The pre-action commit that earlier releases performed in
`onStartCommand`, and the `startForegroundSync` re-post that
`onPlaybackStarted` and `onPlaybackResumed` issued after
`broadcastUpdate`, have both been removed. Either showed a state the
user had already left, and on some devices the two posts interleaved
and produced a brief flash of the pre-tap icon after the post-tap
one. A transport tap now produces exactly one notification commit,
fired by `broadcastUpdate` when the state machine has reached the
new state. `broadcastUpdate` also posts the `PlayerEvent.Snapshot`
sticky event that every activity consumes, in the same turn, with
the same payload.

### Decisions made on the user's behalf

The app makes a handful of decisions on the user's behalf. It makes
them quietly, and once.

The volume receiver is registered when the player binds. Its presence
*is* the session. A service started by a Bluetooth connect is a
service the user did not open, and on such a service the receiver does
not exist — so a hardware volume key just adjusts the volume.

The volume handler reads the current stream volume at the moment it
runs, not the value the broadcast carried, because some devices
deliver the volume-change broadcast late.

A hardware volume press while the player is backgrounded does not
take audio focus from an app that is playing. The raise path, the
Bluetooth volume remap, and the media-session play callback all
require the service to already hold focus; an interruption record
without focus names an interruption the user has not overridden, and
the volume key does not act on it.

When another app asks to share the output, the music ducks rather than
pauses. The duck is applied to the `MediaPlayer`'s own output, not the
system stream. The volume slider does not move.

When another app takes the output away entirely, the service pauses
and remembers that it paused itself for a reason that will end. When
focus returns, the music resumes.

When the user chooses a theme, every activity recreates on the turn
the dialog dismisses. Not on the next launch. A theme change made
from a dialog closes every dialog on the screen on the same sticky
event, so no surface survives the change un-rebuilt.

When the user chooses a language, every activity rebuilds around it on
the turn the dialog dismisses. Not on the next launch.

### Document the why

Every method carries a KDoc that explains not only *what* it does
but *why it exists*. The service class documentation includes a
section called *Volume Semantics* and another called *Audio Focus
Semantics*, and each reads like a decision that was made once and
defended forever. The `MetadataWriter` exposes a `WriteResult.Status`
enum with eight values, each one a distinct outcome the writer can
produce, and the caller is expected to branch on the value, not on the
presence of an error string.

---

## Why the heart is large

`MusicService` is several thousand lines. `MusicPlayer` is several
thousand more. This is a consequence of a decision, not an accident.

A large class is a problem when it fails one of four tests:

**One — it has no single sentence.** If you cannot say what the class
is for in one sentence, the class is not large; it is several.
`MusicService` has a sentence. It is the playback state machine.
Every field in it — the two monotonic counters, the two-index model,
the shuffle history, the shuffle peek memos, the cached position, the
recovery counter, the skip loop budget, the start token, the deferred
reconcile flag, the audio-focus flags, the interruption record, the
two focus requests — exists because that sentence needed it. Not one
is idle.

**Two — it has more than one writer.** Every resource in this app has
exactly one owner. If `MusicService` were split, the playback state
machine would acquire a second writer the moment two of the new
classes both needed to touch `mState`.

**Three — it cannot be held in one head.** This is the only honest
objection to a monolith, and it is subjective. The person who holds
`MusicService` in his head built it one organ at a time, over months.
He has answered this question by continuing to work in this file.

**Four — the boundaries are obvious and the coupling is not.** If the
seams are clear, extracting them is cheap. If the seams are not clear,
extracting them *creates* the loss. In `MusicService` the coupling is
the function. The volume receiver is bound to the session. The session
is bound to the service lifecycle. The service lifecycle is bound to
the state machine. The state machine is bound to the audio-focus
record, which is bound to the media-session's foreground projection.
There is no clean cut that does not pass through tissue.

`MusicService` fails none of these four. It is large because it is one
thing.

`MusicPlayer` — the brain — is a different case. It is the surface
that owns the screen, which means it holds whatever the screen needs.
That is what the brain does. The question is not whether it is large;
it is whether its arrangement is the arrangement its function
required. On that question the answer is not yet obvious.

---

## Known gaps

**No unit tests.** The position is that the only test that matters is
actual use. That position finds every flaw a user will encounter and
none that only a maintainer will.

**Language quality is an ongoing editorial concern.** All sixteen
supported locales carry the same resource contract as the English
master. A native speaker's review remains valuable for idiom, tone,
and regional terminology.

**`MusicPlayer` is not yet obviously arranged.** Its state, subscribers,
gesture handling, and rendering paths are all in one file. The seams
are visible; the arrangement that would use them is not yet obvious.

**The event bus is the app's internal signalling contract.**
`LlamaEventBus` is the sole entry point for signalling, and the
`com.ark.llama.event` package is the sole home of event definitions.
No consumer may slip a direct `sendBroadcast` back into the tree
without the change appearing in an audit of that package or of any
file that registers a `BroadcastReceiver` — the one remaining receiver
being `MusicService.mMusicVolumeReceiver`, whose trigger is external
to the app. The gate exists so that the migration is not re-absorbed
one convenience broadcast at a time.

**The boundaries become clearer as the system grows.** A volume
receiver has meaning only in relation to the listening session. A
`PlayerEvent.Snapshot` has meaning only when its state belongs to the
transition that produced it. Each new capability makes those
relationships more explicit, and the architecture continues to be
evaluated against the way the player is actually used.

---

## A note on the authorship

Llama Music Player was built by two: a developer in Ghana who cannot
write code and a language model that cannot know what a music player
should feel like in a hand. The developer made every design decision.
The model wrote every token. Neither could have made the app alone.

The body the app is shaped as is not a metaphor the developer reached
for. It is the same principle he was trained in as a Physician
Assistant: every structure in a living thing is a design that survived
because the function it served was necessary. He applied the principle
to software because it is the principle he already had.

The full story is in `ORIGIN_STORY.md`.

---

*This document describes the codebase as it is. The implementation is
the authority for runtime behaviour; this document explains the
structure and the decisions behind it.*