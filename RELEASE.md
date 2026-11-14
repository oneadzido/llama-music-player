# Llama Music Player v1.5.0

**Release date:** November 14, 2026

Version 1.5.0 is an architecture release. The app's cross-component
signalling moves from Android broadcast intents to the EventBus, the
source tree is now fully Kotlin, and the build moves to a new
toolchain. No user-visible behaviour changes. Playback, metadata
editing, folder management, theme and language switching, the
equalizer, the notification, the album-art carousel, and the home
gesture all behave exactly as they did in v1.4.8. For the full
change history, see the `[1.5.0]` entry in the
[CHANGELOG](CHANGELOG.md). For the current README — the technical
profile, the installation steps, the permission list, and the
"Inside the codebase" tour — see [README.md](README.md).

---

## What's in this release

### The app signals through the EventBus

Every internal signal the app produces — a transport command, a
playlist lifecycle event, a metadata update, a theme or language
change, an audio-session notification, a playback snapshot — used to
travel between components by one of two mechanisms: a direct method
call inside the app's own objects, or an Android broadcast for the
deliveries a method call could not make. The broadcast half of that
pair is gone. Every internal signal now travels on the process
EventBus, and every consumer subscribes to the event it needs.

Twenty-nine internal `sendBroadcast` calls are removed, five
`BroadcastReceiver`s that existed only to consume them are removed,
and thirteen action constants whose only readers were the removed
broadcasts are removed. `VOLUME_CHANGED_ACTION` is retained: it is
the system broadcast the framework emits when the user presses a
hardware volume key, and `MusicService` is a consumer of the system,
not a producer of an internal signal. The five notification
transport actions the notification's buttons dispatch to
`MusicService.onStartCommand` are also retained — they are the
`PendingIntent` actions on the notification, not app-internal
broadcasts.

The new `LlamaEventBus` façade (`com.ark.llama.event.LlamaEventBus`)
is the single entry point. No consumer touches
`EventBus.getDefault()` directly. `LlamaEventBus.init()` is called
once, at the top of `LlamaMusicPlayer.onCreate`, before any manager
whose constructor registers a subscriber. The bus is built with the
library's reflective subscriber lookup. The one `@PublishedApi
internal` constant is `TAG`, referenced by the two public inline
functions `latest<T>()` and `removeLatest<T>()`; Kotlin rejects a
public inline function that reads a non-public member, and
`@PublishedApi` opens exactly that one reference.

### The source tree is fully Kotlin

Every source file under `app/src/main/kotlin/com/ark/llama` and
`app/src/main/kotlin/com/ark/llama/event` is Kotlin. The
`app/src/main/java/` directory is absent — the migration moved every
file and left the directory empty. The CI workflow's "Verify
Kotlin-only source layout" step is a gate: it fails the build when a
`.java` file appears outside the build output, or when `.kt` files
have drifted out of `app/src/main/kotlin`.

The build is now compiled with Kotlin 2.0.21 under AGP 8.7.2 and
Gradle 8.9, with the Kotlin and Java outputs byte-compatible at the
17 level. The Kotlin DSL for the two Gradle build files and the
Kotlin source tree were already in place in v1.4.8; v1.5.0 completes
the move by removing the last Java-adjacent paths from the source
layout and by gating future drift in the CI workflow.

### The toolchain is upgraded

`AGP 8.2.2 → AGP 8.7.2`, `Gradle 8.x → Gradle 8.9`,
`Java 17 → Java 17 (source, target, and JVM target)`, and
`compileSdk 35 → compileSdk 36`, `targetSdk 35 → targetSdk 36`. The
`buildToolsVersion` is pinned to `36.0.0`. The AGP-to-Gradle pair is
compatible as a pair — AGP 8.7.2 requires Gradle 8.9 or newer — and a
future bump must move them together.

`compileSdk = 36` uses AGP 8.7.2, which was tested up to
`compileSdk = 35`. The mismatch is suppressed by
`android.suppressUnsupportedCompileSdk=36` in `gradle.properties`.
API 36 is backward compatible with the API 35 surface the app uses,
and the app uses no API 36-only features. The bump to AGP 8.9 is
deferred to a future release when the AGP and Gradle versions can be
moved together.

The Kotlin `jvmTarget` and the Java `sourceCompatibility` and
`targetCompatibility` are all 17, so the Kotlin and Java outputs are
byte-compatible at the 17 level. Core library desugaring is enabled
for the `java.time` API surface that jaudiotagger relies on.

### A new Application class installs the process-wide singletons

`LlamaMusicPlayer` (the Application, not the player screen) installs
the process-wide singletons in dependency order before any activity
or service runs: the EventBus first, then the language manager (whose
localised context every later manager reads), then the character
mapper (which builds the recognised-placeholder set from every
supported locale), then the theme manager, the icon font, the album
art cache, the SAF helper, the equalizer, the notification
controller, the toast and overlay stacks, the playlist manager, and
finally the navigation controller. A cold start that reaches
`MusicPlayer`, or a headset connect that starts `MusicService` from
a cold process, both find the bus installed and the managers
constructed before their first interaction with either.