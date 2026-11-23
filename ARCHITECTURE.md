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
`MusicService.handleAudioFocusChange(int)`. The process-wide broadcast
registration has one writer, `BroadcastTranslator.start()`.

**One shape per role.** A class that plays a role — singleton, state
machine, ticker, safely-suffixed helper, state listener — has the
same shape it has in every other class. The role determines the shape,
not the class. Every singleton with no construction-time side effects
uses the same double-checked locking pattern. Every safely-suffixed
helper takes a nullable parameter, returns early on null, catches
`Exception`, and writes the reason in a comment. Every greenrobot
subscriber uses the default `ThreadMode.POSTING` and expects to run
on the main thread.

**One name per concept.** The same concept carries the same name in
every file. `mMainHandler` is the handler bound to the main looper.
`getInstance` is the singleton accessor. `*Safely` is the "swallow
the exception, the failure is not interesting" suffix. `*Event` is
the suffix for a typed event posted through `BroadcastTranslator`.
There is no second name for any of these.

**Document the why, not the what.** Javadoc explains the reason a
method exists, the invariants it holds, and the failure modes it
guards against. The method body shows the what; the javadoc shows
the why.

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

An empty playback scope is a real state, not an empty-looking playlist
that still owns a hidden current song. Entering it tears down the active
player and clears current and pending indices. The service retains only
the prior song path under a separate scope-resume preference. When a later
scope becomes non-empty, that path is selected if it belongs to the new
scope; otherwise the first song in the scoped playlist is loaded. The
retained path is cleared only after a replacement song successfully
prepares, so a failed preparation does not lose the restoration choice.

The heart has one method that expresses the display-song rule. It
is not a two-line accessor; it is a guarded evaluation of "which song
should the carousel be sliding toward" that returns null when the
answer is not yet available.

```java
@Nullable
private Song getDisplaySong() {
    if (mCurrentPlaylist == null || mCurrentPlaylist.isEmpty()) {
        return null;
    }
    if (mState == ServiceState.PREPARING) {
        if (mTargetIndex < 0
                || mTargetIndex >= mCurrentPlaylist.size()) {
            return null;
        }
        Song targetSong = mCurrentPlaylist.get(mTargetIndex);
        if (targetSong == null
                || TextUtils.isEmpty(targetSong.getPath())) {
            return null;
        }
        return targetSong;
    }
    if (mCurrentIndex >= 0
            && mCurrentIndex < mCurrentPlaylist.size()) {
        Song currentSong = mCurrentPlaylist.get(mCurrentIndex);
        if (currentSong == null
                || TextUtils.isEmpty(currentSong.getPath())) {
            return null;
        }
        return currentSong;
    }
    return null;
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
their current surface, rather than to clear it. `snapshotForNotification()`
returns null, and both `commitFromBroadcast` and `syncOnce` refuse
to render that null, leaving the notification on its last rendered
snapshot. The carousel holds its last window. The player UI holds
its last drawn title and artist. The surface clears only when the
state machine itself transitions to `IDLE`, which the controller
checks through `ServiceStateSource.isIdle()`.

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

Metadata editor announcements are consumed independently by
`MusicService`, `MusicPlayer`, and `PlaylistActivity`. The service applies
announced fields to matching entries in its full and current playlists.
For each text field, `null` means unchanged, an empty string means an
explicit clear, and a non-empty value replaces the stored value. The
service uses raw song values for fields not present in a partial update,
so localized display placeholders do not become persisted metadata.
Artwork follows a separate three-state contract: `null` means unchanged,
an empty byte array means removed, and non-empty bytes mean replacement.
The player invalidates its album-art carousel path only when artwork
actually changes, and refreshes the title and artist when the edited song
is current. The playlist replaces and notifies only the affected row and
updates its playing indicator only when that row exists.

The scope itself is `PlaybackScope`: an immutable value with seven
kinds — All Songs, Favorites, Loose Tracks, Specific Folders,
Specific Genres, Specific Artists, Specific Albums — and four filter
sets, of which exactly one is populated for the four filter kinds.
`matches(Song, Set)` is the only read; it consults
`CharacterMapper.isUnknownX(String)` so a song whose field is the
current locale's placeholder is never matched against a stored
value. Loose-track and selected-folder comparisons pass filesystem
paths through `PlaybackScope.normalizePath()` first. The normalizer
standardizes slash direction, removes redundant `.` and `..` segments,
and maps common primary shared-storage aliases to
`/storage/emulated/0`. It is lexical only: it does not resolve symbolic
links or change path case. Folder matching still checks the separator
boundary after the selected folder, so sibling names with a shared
prefix are not included. A malformed serialized scope deserializes to
`PlaybackScope.ALL_SONGS`, so a corrupted preference cannot leave
the player with a scope that matches nothing. `MusicService` persists
the active scope under `playback_scope_json` and exposes it through
`getPlaybackScope()` and `setPlaybackScope(PlaybackScope)`; the latter
is the single writer of the fact.

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
`FolderManager` broadcasts it under `APPLY_PLAYBACK_SCOPE`. The
service is the single writer of the active scope.

Every playback destination is evaluated against `mScopedPlaylist`, not
the grace-period-extended `mCurrentPlaylist`. The grace-period entries
keep the current and pending songs alive while a scope changes; they do
not become candidates for manual Next or Previous, automatic completion,
prepare-failure recovery, player-error recovery, or stall recovery.
`computeNext()` and `computePrevious()` resolve a path in the scoped list
and map the selected `Song` back to its current-playlist index, preserving
the state machine's existing index model and transition owner.

Natural completion is also a scoped destination. `onCompletion()`
resolves the next song from `mScopedPlaylist`, not from the
grace-period-extended `mCurrentPlaylist`. A scope change that returns
to All Songs — for example, from Favorites — therefore advances into
All Songs on completion, rather than stalling because the completed song's
position was resolved against the old scope.
`reconcilePlaybackAfterScopeChange()` keeps the current and pending
songs aligned with the new scope before completion runs, so the completion
path never reads a stale index.

The two-index model carries one invariant the completion path depends
on: a handoff is only real while `PREPARING`. While `PLAYING`, the
target must equal the current song. The reason is that `onCompletion()`
fires only once. If it returns for a stale target, playback sits silent
at end-of-song in `PLAYING`, with the position already at the duration,
and the health check cannot save it — there is no stall left to detect,
only a finished song that will never fire again. Any rebuild while
`PLAYING` — a scope, favorite, sort, or `setPlaylist()` rebuild — can
leave `mTargetIndex` pointing at an old-scope or grace-period song, so
the invariant is re-established, not assumed. `applyPlaybackScopeInternal()`
and `reconcilePlaybackAfterScopeChange()` align a stale target to the
current song while `PLAYING`, and `onCompletion()` itself aligns a
stale target before resolving the next song and never suppresses
completion for one. `onCompletion()` also uses the `scopeEmpty()`
helper to enter the no-song state when the scope has no eligible songs.

The carousel's previous/next neighbour preview uses the same scoped
candidate set. Shuffle preview and commit share the same memoised choice,
and the shuffle history stores and returns only scoped destinations.
When a scope change leaves the current song outside the new scope, forward
navigation enters the first scoped song and backward navigation enters
the last scoped song; the out-of-scope grace-period entry cannot leak into
subsequent navigation. Repeat off advances within the scope and pauses
when the current song is its final entry. An empty scope stops playback
and clears the current queue rather than pausing an excluded song. Repeat
all wraps within the scope, shuffle selects from it, and repeat one
continues the current song only while that song belongs to the active
scope. If it does not, completion or manual navigation enters the new
scope instead of repeating the excluded song. Applying an empty scope is
a stronger state transition: `MusicService.reconcilePlaybackAfterScopeChange()`
releases the player, clears current and target indices, empties the
transitional current playlist, clears playback artwork and notification
state, and publishes IDLE with a zero scoped-song count. The UI's no-song
state therefore corresponds to actual service silence, not placeholder
labels over a still-playing excluded track. If the new scope has eligible
songs, an excluded current song is replaced by a scoped song; an in-flight
prepare targeting an excluded song is invalidated before it can commit.
Favorite changes that remove the current song from a Favorites scope use
the same reconciliation. Scope-only broadcasts receive a newer generation
when no prepare is in flight, so delayed updates cannot overwrite a newer
display.

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

A transition creates a new player, and a new player starts with
default `PlaybackParams`. The heart therefore does not assume the
new player inherits pitch and speed. `onPrepared()` attaches
through `EqualizerManager.attachToAudioSession()`, which snapshots
and restores the current equalizer state across the session change,
and playback parameters are applied on the player thread before
`start()` and in the READY branch, which never reaches
`onPlaybackStarted()`, instead of leaving the prepared player on
defaults until a resume.

### The lungs

`PlaylistManager`.

The lungs breathe life into the playlist. Folder scans, genre loads,
and library resets are all `PlaylistManager`'s job, and each of the
three is an operation on the `NavigationController` stack that
declares its own liveness window. The reset operation has a write
barrier, an invalidation broadcast, and a completion signal, and its
structure follows four rules: one owner, one write barrier, one
invalidation broadcast, one completion signal.

In 1.5.0 the lungs subscribe to their own event. The
`TRIGGER_GENRE_LOADING` stem, which the lungs used to receive
through a receiver, is now a `TriggerGenreLoadingEvent` posted by
`BroadcastTranslator` and consumed by a single `@Subscribe` method
on the manager. The subscribe/unsubscribe pair is wired to the
singleton's lifecycle — the private constructor and `shutdown()` —
rather than to an activity lifecycle, because the manager is not
an `Activity`.

### The nervous system

`MusicPlayer`, `BroadcastTranslator`, `NavigationController`.

The nervous system has three parts.

`MusicPlayer` is the brain — the main surface, coordinating
everything the user sees. It is the largest class in the codebase,
and the next section explains why that is a decision rather than an
accident.

`BroadcastTranslator` is the spinal cord. The spinal cord is not
the brain's assistant; it is the trunk that every nerve in the body
runs through. A signal that begins in the brain, a signal that
begins in the skin, a reflex from the fingertip — all of them
travel through the spinal cord to reach where they need to go. No
part of the body reaches another part without passing through it.
`BroadcastTranslator` plays that role for the app. It is not the
signal itself, and it is not the coordination logic — that lives in
the senders and the subscribers. It is the trunk those signals
travel along: registered once at process start, one receiver per
stem, one typed event on the way out, one subscriber per event on
the way back. Every in-process broadcast the app uses goes through
it, so no class needs to know where the other classes are. In-process
event delivery is provided by greenrobot, delivered through
`BroadcastTranslator`.

`NavigationController` is the peripheral nerves. Where the spinal
cord carries every signal along a shared trunk, a peripheral nerve
carries a specific action out to the specific classes that listen
for it. `NavigationController` enforces *declared liveness*. Each
operation announces how often it will heartbeat and how many missed
heartbeats it tolerates. The controller does not guess. It reads
the declaration and enforces it. An operation that heartbeats
every 500 milliseconds and tolerates three misses has a
fifteen-hundred-millisecond window, and the controller does not
need to know why the window is that size. It is a nerve, not a
trunk: a specific signal for a specific target.

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

### The senses

`EqualizerManager`, `StorageObserver`, `CharacterMapper`.

`EqualizerManager` is the ears. It owns the audio effect objects and
the persisted settings. SharedPreferences are the authoritative store;
getters read what setters wrote, regardless of any rounding the
underlying effect applies. The band label above each slider reads its
unit symbol from a resource, so a Russian device shows `Гц` and every
other device shows `Hz`.

Pitch and speed are user preferences, not per-track state. They
live in the same persisted store as the bands, and they survive a
session change for the same reason the bands do: attaching to a new
audio session rebuilds the effects for that session, but must not
rewrite what the user set. `attachToAudioSession()` therefore
snapshots and restores the current preset, band levels, bass boost,
surround, pitch, and speed, so a later read returns the user's
values, not defaults.

`StorageObserver` is the listening. It registers a single content
observer for the audio MediaStore URI. Audio additions, removals, and
metadata edits trigger a quiet period and then a scan. Image, video,
and download writes do not.

`CharacterMapper` is the reading. When a tag is absent, unreadable, or
carries a recognized placeholder value, the parser produces a
canonical value in the app's current language. The value is resolved
from a string resource at the moment it is needed, so a German library
shows the German placeholder for a song whose artist tag is missing,
and an Arabic library shows the Arabic one.

### The reflex arc

`BluetoothReceiver`.

A reflex arc is a small circuit — a sense, a nerve, a muscle, no
brain in the loop. `BluetoothReceiver` is exactly that. When a headset
or A2DP profile connects or disconnects, the receiver broadcasts a
pause request that `MusicService` consumes. The service owns the
pause because the service owns the audio, and the pause fires whether
or not the player activity is alive. The receiver stays a manifest
receiver — it is a system broadcast that fires when the app is not
running — and the service's consumption of its pause request is now
a `BluetoothPauseEvent` posted by `BroadcastTranslator`, not a
receiver the service owned directly.

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
files.

### The voice

`ToastManager`, `NotificationController`.

The voice speaks in two registers. `ToastManager` is the voice inside
the app, holding an operation stack. An operation is pushed with an
identifier, and the stack renders the top operation's message. A
lower operation is retained but not shown. Cancelling an operation is
removing its entry, which is indistinguishable from an operation that
completed. The stack does not pretend to know what it cannot know;
callers that need to distinguish those two cases carry their own
terminal state.

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
persisted playback state and re-seeds the playlist behind it:
the full library is loaded first, then exactly one playlist call
installs it - or just the last song when the load fails or does
not contain it - and the last song is prepared to `READY`,
without starting playback, so the notification's Play, Next,
and Previous buttons work without the activity opening, and the playlist
is not replaced while a restore prepare is in flight. A
restart that finds a playlist already seeded does not re-seed
it or overwrite the live notification with a paused snapshot. While the service is reconciling a prepare
failure or player error, its pending-transition state prevents the
notification's cancel path from removing the visible playback surface.

The controller reads its rendered content from the service through
`ServiceStateSource.snapshotForNotification()`, which calls
`getDisplaySong()` unconditionally. The notification therefore shows
the target during a transition, the same song the carousel and the
player screen are showing, and advances with them on the same frame.

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

The voice speaks through one channel registry. Both `ToastManager`
and `NotificationController` subscribe to `THEME_COLOR_CHANGED`
through `BroadcastTranslator`, not through receivers they register
themselves. The translator is the spinal cord every in-process
signal travels along; the classes that read those signals are
`@Subscribe` subscribers. A theme change reaches `ToastManager` as
a `ThemeColorChangedEvent` and reaches `NotificationController` as
the same event type. `ToastManager` answers by rebuilding its own
surface from the persisted theme on the next commit.
`NotificationController` takes no action on the event itself:
`MusicService` forces the commit on the same turn (see The
skin), so the notification is rebuilt from the persisted theme
immediately rather than waiting for the next playback change.

### The skin

`ThemeManager`, `ThemeHelper`.

`ThemeManager` is the skin's color. `ThemeHelper` is the skin's touch,
painting every control. One value drives every button, every seek bar,
every cursor, every tinted placeholder, and the notification's own
large icon. The theme's *name* is a resource, so a German theme list
says "Königliches Lama" and an Arabic one says "لاما ملكي", and both
refer to the same color value in the same array.

A theme change is a rebuild, not a repaint.
`ThemeManager.setThemeColor(int)` persists the value and posts
`ThemeColorChangedEvent` through `BroadcastTranslator`; every
activity answers with `recreateActivity()`, every open dialog
dismisses itself, and every surface is re-constructed against the
persisted color. `ThemeHelper` still paints each newly-constructed
view; it no longer has a live view tree to repaint on a broadcast,
and the theme's color is read at construction rather than patched
in. `LanguageManager.setLanguage(String)` posts
`LanguageChangedEvent` through the same translator, and every
subscriber answers with the same `recreateActivity()`. The two
stems share one delivery mechanism, one dismissal order, and one
handler. The broadcast is a signal to rebuild, not an instruction
to patch.

A theme event that arrives while an activity is not visible is
deferred, not dropped. The handler records it in
`mThemeRecreationPending` and the recreate runs when the activity
returns — from `onResume()` for the activities whose guard is
the resumed gate, from `onStart()` for `MusicPlayer`, whose
guard is the started gate. A backgrounded activity therefore
still rebuilds against the new theme instead of keeping the
theme it was constructed with. Language is not deferred: the
language picker is hosted by a visible activity, so there is no
backgrounded-activity drop to defer, and every subscriber
recreates immediately.

The selected locale is reapplied after activity resume and
configuration changes, and timer digits are formatted with an
explicit formatter for the selected locale, so a resumed or rotated
activity does not fall back to the system locale.

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

A tap on a playlist row does not highlight from the stale displayed
list. The tap sends `PLAY_SONG`, then the activity refreshes first —
`refreshPlaylistFromService()` in the delayed block, before it computes
the highlight — re-reading the service's current playlist after the
service's `rebuildPlaylists()` on commit. The refresh is the evaluation:
songs no longer in scope and no longer current/target are evicted, the
tapped song survives as the grace-period current/target when it
committed even if it is out of scope, and a tapped song that never
became current is not highlighted as playing. If the prepare commits
after the tap's own refresh, the `UPDATE_PLAYER` broadcast does the
same work again: `onUpdatePlayerEvent()` refreshes when the playing
song is not in the displayed list, then highlights from the fresh
list. The refresh still respects the genre-loading and scrolling
suppression, and it adds no new evaluation to the service — the
service already rebuilt; the activity only asks for the result.

Hiding search follows the same rule. `hideSearch()` restores the
full list, then focuses the current song with the same delayed,
guarded focus the sort path uses, so the scroll runs after the
adapter has laid out the restored list and is suppressed while the
user is scrolling, tapping, or genre-loading.

`EqualizerActivity` is the equalizer — ten bands and sixty presets,
with every seek bar snapping to a step under a guard flag so a
programmatic re-set does not re-enter the listener that performed the
snap. The reset button reports *no change* when nothing has moved.

Attaching to a new audio session preserves the current equalizer
state for the same reason it preserves the bands: preset, band
levels, bass boost, surround, pitch, and speed are user preferences,
not per-track state. `EqualizerManager.attachToAudioSession()`
snapshots and restores them, so the refresh that follows a transition
reads the preserved values into the SeekBars.

`FolderManager` is the folder organ — speaking Storage Access
Framework, taking a persistable permission, mapping the URI to a
filesystem path, and cleaning up the loose tracks when a folder is
removed. Its header carries the SCOPE button, which opens the
playback-scope dialog; the dialog broadcasts the serialized scope to
the service through `APPLY_PLAYBACK_SCOPE`, so `FolderManager` itself
does not bind `MusicService`.

### The face

`CreditsDialog`, `LanguageDialog`, `PresetDialog`, `ThemeDialog`, `PlaybackScopeDialog`.

The face is what tells the user what the app is. `CreditsDialog` is
the front of it. Its version header carries the VERSION label, the
language button, and the version number. `LanguageDialog` is the
speaking face — picking the language. `PresetDialog` is the
equalizer's face. `ThemeDialog` is the skin's face.

Four dialogs share one top-bar shape: the Cancel button at the start
with its natural width, the title in the middle with a weight of 1,
and a trailing placeholder that reserves no width. The title is the
only element that can yield, so a long translation truncates with an
ellipsis rather than running under a button beside it.

All five dialogs share one window-context rule: the dialog window
is attached to the original Activity context, while dialog views
are built from the locale-wrapped context, wrapped in the app locale
before view creation. A locale wrapper is not an Activity and
carries no window token, so a dialog built on it has no valid
window.

A theme change made from `CreditsDialog`'s picker closes every dialog
on the screen — the picker, the `ThemeDialog` it opened, and the
`CreditsDialog` underneath — on the same event the hosting activity
recreates on. `ThemeDialog` and `LanguageDialog` fire their listeners
from their own `OnDismissListener`, so the picker's window is already
detached before the host activity's `recreateActivity()` runs. There
is no window in which one dialog has been rebuilt and another has
not, because no dialog is rebuilt in place; all of them are
dismissed, and the next open constructs a new dialog against the
persisted theme or locale from the first frame.

`PresetDialog` is the one dialog whose host does not wait for a
notification. Picking a preset applies it through the manager,
and the host then refreshes the band controls from the manager's
current state on the same turn — the dialog owns the selection,
the host owns the refresh, from the source of truth, rather
than from a listener notification that may not arrive.
Reselecting the preset the dialog opened with applies nothing,
the same return the theme and language hosts give a reselect:
the dialog closes with its bounce and the host returns before
the operational checks. The guard lives in the host listener,
not in the dialog, which still fires for every tap.

`PlaybackScopeDialog` keeps one draft set per filter kind while
it is open. A mode switch changes only the selected kind and
the expansion flag; it never clears a set, so returning to a
kind restores its checks exactly as they were left. Nothing
commits on a mode switch — the committed scope is built from
the selected kind's set alone, and Cancel / dismiss commits
nothing. The drafts die with the dialog; reopening starts
from the applied scope. On open the dialog scrolls to the
current scope when it is not already in view: the first
checked filter of the current kind, or the kind's mode row
when it has no checked values. The scroll runs again once
the genre, artist and album lists finish loading, so a
checked filter that did not exist at open still lands in
view. A target already in view never scrolls, and the
scroll is an instant jump, not an animation.

Screen rotation is a decision, taken in the manifest
alone: every activity declares
`screenOrientation="unspecified"` — the platform value
for auto-rotation, so the system decides the orientation
honouring the user's auto-rotate setting
— and `orientation` is in the shared configChanges set,
so rotating neither destroys the activity nor dismisses
an open dialog. The activities
that own size-dependent layout re-apply it from
`onConfigurationChanged` — the equalizer split layout, the
folder-manager layout, the player album-art size — keyed
off the new window height, and every dialog window is
`MATCH_PARENT`, so it re-measures with the window. That is
what makes rotation safe for long locale labels: the row
simply gets wider.

Lifting the lock exposed one overlap: in a short landscape
window the player's album art is gone, the weight-1
song-info group collapses to zero height, and its children
spill over the seek bar, because the root does not clip
children. `MusicPlayer.applySongInfoLayout` therefore docks
the group beside the transport row when the art is hidden,
the orientation is landscape, and the window is at least
600dp wide (`DOCK_MIN_WIDTH_DP`): the two views are
re-parented into a horizontal row — group first, then the
transport row — sharing the row with 0.5 weight each. The
group keeps its normal padding and gravities, and the
transport row keeps its four spacers, so it looks like its
portrait version. The group starts 16dp from the left edge
and the controls end 16dp from the right edge, so the gap
between the two sits at the center. The docked row hugs its
content and carries a 48dp bottom margin: the toast lane,
reserved between the side-by-side row and the bottom buttons.
The seek-bar section takes what is left — weight 1, its content
centred — because the side-by-side row, the toast lane, the
bottom buttons and the 16dp to the edge are fixed. Rotating
back restores both views to their own rows, with their original
indices and layout params. The toast lane is reserved in both
orientations: 48dp in split view and in landscape, plus the bottom
buttons row's 16dp top padding. In split view, the 48dp is the
transport row's bottom padding; in landscape, it is the docked
row's bottom margin, and the transport row's bottom padding
stays zero so the lane is the margin, not padding. In portrait with
the art visible, the lane is the layout's 64dp transport padding
instead. The 16dp from the bottom buttons to the edge is the
bottom buttons row's bottom padding and is never touched.
The dock only applies in wide landscape. A short portrait
window is too narrow for the two side by side, and so is a
narrow landscape window: landscape split-screen, at about
400dp, would give the transport row about 200dp against the
288dp it needs and leave the title about 56dp after the
group's 72dp + 72dp inner padding. Such a window falls back
to the stacked portrait layout instead, group on its own
row and transport on its own row, each at full width — but
a short, undocked window cannot fit the whole stacked
layout either, so the seek-bar section is hidden there.
That is the same evaluation in short portrait split-screen
and in short, narrow landscape: `albumArtHidden &&
!dockSongInfo`. The seek bar goes, not the group: the
group is the glanceable part — what is playing, home,
favourite — plus the transport controls, while scrubbing
can wait until the window is full-size again. Hiding the
group instead would leave controls for a song the user
cannot identify, and would take home and favourite with
it, since both live inside the group. The SeekBar keeps
receiving progress updates while hidden, so returning to
a full-size window shows the current position with no
re-sync.
In the stacked layout the group stays vertically centered
in the space between the header and whatever follows it —
the seek bar, or the transport row when the seek bar is
hidden — with its default top padding and its default
gravities. It is never
pinned to the top, and no extra top padding is added: the
old 16dp album-art offset pushed the centered group down
and read as an extra gap at the top. The seek-bar
progress drawable is `seekbar_progress_enabled`, the
enabled-state progress; the name says which state it
draws, the drawable itself is unchanged.

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
with. `LanguageManager` is the tongue.

Every user-visible string the app shows is a resource, and the set of
supported languages is declared three times:

- `LanguageManager.LANGUAGE_TAGS` — the app's own list
- `res/xml/locales_config.xml` — the platform's per-app language picker
- `app/build.gradle`'s `resourceConfigurations` — the release
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

Every in-process event posted by `BroadcastTranslator` uses the
default `ThreadMode.POSTING`, so a subscriber runs on the same
main-thread turn the broadcast it consumes was originally
dispatched on. Subscriber bodies are byte-for-byte the
`onReceive` bodies they replace; they assume they are on the main
thread for the same reason the receivers did.

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
A cold-start `setPlaylist` is a commit turn too: the ordinary
cold-start branch prepares the last track and position into `READY`
without autoplay, regardless of whether playback was active before
the process died, and commits with `broadcastUpdate()` after
`reconcile()`, so the transport controls land on the same turn.
Only explicit pending-song launches may autoplay.
The pre-action commit that earlier releases performed in
`onStartCommand`, and the `startForegroundSync` re-post that
`onPlaybackStarted` and `onPlaybackResumed` issued after
`broadcastUpdate`, have both been removed. Either showed a state the
user had already left, and on some devices the two posts interleaved
and produced a brief flash of the pre-tap icon after the post-tap
one. A transport tap now produces exactly one notification commit,
fired by `broadcastUpdate` when the state machine has reached the
new state.

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
from a dialog closes every dialog on the screen on the same
broadcast, so no surface survives the change un-rebuilt.

When the user chooses a language, every activity rebuilds around it on
the turn the dialog dismisses. Not on the next launch.

### Document the why

Every method carries a javadoc that explains not only *what* it does
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

**`MusicPlayer` is not yet obviously arranged.** Its state, receivers,
gesture handling, and rendering paths are all in one file. The seams
are visible; the arrangement that would use them is not yet obvious.

**The boundaries become clearer as the system grows.** A volume
receiver has meaning only in relation to the listening session. A
broadcast has meaning only when its state belongs to the transition
that produced it. Each new capability makes those relationships more
explicit, and the architecture continues to be evaluated against the
way the player is actually used.

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