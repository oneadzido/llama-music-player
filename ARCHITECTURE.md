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
`AlbumArtCache`.

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

**Document the why, not the what.** Javadoc explains the reason a
method exists, the invariants it holds, and the failure modes it
guards against. The method body shows the what; the javadoc shows
the why.

---

## The systems of the body

### The heart

`MusicService`, `PlaybackWindow`, `AlbumArtCarousel`.

The heart is `MusicService`, and its job is to pump audio through the
app. A single `ServiceState` enum, two monotonic counters, and a
two-index model (`mCurrentIndex` and `mTargetIndex`) hold every
transition-related value. State mutations run on the main thread; raw
`MediaPlayer` calls run on a dedicated `HandlerThread`. Only
marshalled `Runnable`s cross between them.

The heart has one method that expresses the app's central rule. It
is not a two-line accessor; it is a guarded evaluation of "which song
is the app showing right now" that returns null when the answer is
not yet available.

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

During a transition, the display song is the target. Otherwise it
is the current song. Every consumer of "which song is the app
showing right now" — the notification, the media session, the
carousel, the player UI, the art prefetch — reads this method.
There is no second expression of the rule.

The guards are the method's contract. The playlist can be empty
during a library reset. `mTargetIndex` can point past the end of
the playlist during the window between a `setPlaylist` replacement
and the next reconcile. The song at the index can be null during
a database read that has not yet completed. The display song's
path can be empty during a loose-track insert that has not yet
written the row. Every one of those cases returns null, and every
consumer of the method treats null as "nothing to render" rather
than as an error. The notification stays on its last rendered
snapshot. The carousel stays on its last window. The media
session's metadata is cleared.

The consequence of the rule is visible in the app. Tap Next, and
the notification title changes before the audio does. The
lock-screen carousel moves with it. The album art on the player's
own screen begins to slide. All three say the same thing, at the
same moment, because all three asked the same method.

The blood the heart pumps is `PlaybackWindow`: an immutable value
holding three paths — the previous neighbour, the current song, the
next neighbour — plus the service's monotonic transition counter. The
service computes it; the carousel consumes it. The carousel holds no
playlist, no index, no bitmap as primary state.

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

### The lungs

`PlaylistManager`.

The lungs breathe life into the playlist. Folder scans, genre loads,
and library resets are all `PlaylistManager`'s job, and each of the
three is an operation on the `NavigationController` stack that
declares its own liveness window. The reset operation has a write
barrier, an invalidation broadcast, and a completion signal, and its
structure follows four rules: one owner, one write barrier, one
invalidation broadcast, one completion signal.

### The nervous system

`NavigationController`, `MusicPlayer`.

`NavigationController` is the nerves that coordinate actions across
the app. It enforces *declared liveness*. Each operation announces how
often it will heartbeat and how many missed heartbeats it tolerates.
The controller does not guess. It reads the declaration and enforces
it. An operation that heartbeats every 500 milliseconds and tolerates
three misses has a fifteen-hundred-millisecond window, and the
controller does not need to know why the window is that size.

`MusicPlayer` is the brain — the main surface, coordinating everything
the user sees. It is the largest class in the codebase, and the next
section explains why that is a decision rather than an accident.

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
a single memory: one song and its facts.

### The senses

`EqualizerManager`, `StorageObserver`, `CharacterMapper`.

`EqualizerManager` is the ears. It owns the audio effect objects and
the persisted settings. SharedPreferences are the authoritative store;
getters read what setters wrote, regardless of any rounding the
underlying effect applies. The band label above each slider reads its
unit symbol from a resource, so a Russian device shows `Гц` and every
other device shows `Hz`.

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
or not the player activity is alive.

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
written from the same source in the same main-thread turn.

### The skin

`ThemeManager`, `ThemeHelper`.

`ThemeManager` is the skin's color. `ThemeHelper` is the skin's touch,
painting every control. One value drives every button, every seek bar,
every cursor, every tinted placeholder, and the notification's own
large icon. The theme's *name* is a resource, so a German theme list
says "Königliches Lama" and an Arabic one says "لاما ملكي", and both
refer to the same color value in the same array.

### The organs the user touches

`PlaylistActivity`, `EqualizerActivity`, `FolderManager`.

Three activities are the surfaces a user spends most of their time in.
`PlaylistActivity` is the playlist — ten sort modes, a live search, an
edge fast-scroll, and a header that reads `SORT MODE [TITLE A-Z]` or
`SORTIEREN [TITEL A-Z]` or `СОРТИРОВКА [НАЗВАНИЕ А-Я]`, because the
label after the bracket comes from a resource and the space between
the two halves of the phrase is supplied by a Java-side constant.

`EqualizerActivity` is the equalizer — ten bands and sixty presets,
with every seek bar snapping to a step under a guard flag so a
programmatic re-set does not re-enter the listener that performed the
snap. The reset button reports *no change* when nothing has moved.

`FolderManager` is the folder organ — speaking Storage Access
Framework, taking a persistable permission, mapping the URI to a
filesystem path, and cleaning up the loose tracks when a folder is
removed.

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
ellipsis rather than running under a button beside it.

### The fingers

`PlaylistAdapter`, `FolderAdapter`.

The fingers touch the rows. Each adapter renders one list. Neither
holds the data; the activity owns it, and the adapter asks for what it
needs on each bind. `PlaylistAdapter`'s subtitle joins artist, album,
and genre with a bullet, and each field is checked against the
corresponding `CharacterMapper` predicate before it is rendered, so a
field that carries a placeholder from any supported locale renders the
current locale's placeholder in its place.

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

When another app asks to share the output, the music ducks rather than
pauses. The duck is applied to the `MediaPlayer`'s own output, not the
system stream. The volume slider does not move.

When another app takes the output away entirely, the service pauses
and remembers that it paused itself for a reason that will end. When
focus returns, the music resumes.

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
reconcile flag — exists because that sentence needed it. Not one is
idle.

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
the state machine. There is no clean cut that does not pass through
tissue.

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

**The translations are uneven.** The Latin-script and Cyrillic-script
files were produced with high confidence. The Chinese, Japanese,
Korean, Hindi, Arabic, Turkish, and Indonesian files would benefit
from a native speaker's review. The release notes say so.

**`MusicPlayer` is not yet obviously arranged.** Its state, receivers,
gesture handling, and rendering paths are all in one file. The seams
are visible; the arrangement that would use them is not yet obvious.

**Some flaws are only visible after earlier flaws are fixed.** A
volume receiver that fired when the user had not opened the app became
visible the moment a session concept existed. A broadcast whose value
was stale became visible the moment the handler trusted the read
instead of the extra. This is the shape of software, not a flaw in the
process, but it means the list of known gaps is never complete.

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

*This document describes the codebase as it is, not as it should be.
Where the two disagree, the code is the authority and this document is
the correction to make.*