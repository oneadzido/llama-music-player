# Llama Music Player v1.3.4

**Release date:** Oct 15, 2026

### Overview

An internal consistency pass. Sixteen places where the codebase's own
conventions had not been applied were brought into line with the
conventions the rest of the codebase uses. No user-visible behavior
change, with one exception: four dialogs now log a warning when the
app's localized context cannot be reached, in the same way the
managers already do.

### Changed

**Singleton shape.** Four classes were brought into the shape the
codebase's own convention prescribes for their group. `EqualizerManager`
and `PlaylistManager` had been using a synchronized method for classes
with no construction-time side effects; both now use the
double-checked locking pattern the lazy-access group uses.
`ThemeHelper` had been carrying three protections for one invariant —
a volatile field, a synchronized method, and an inner synchronized
block — where the pattern calls for two; the redundant protection is
removed. `OverlayManager` had a volatile field on a class whose
`init` is a synchronized method and whose `getInstance` reads after
the fact; the volatile is removed.

- `EqualizerManager.getInstance(Context)` — double-checked locking with
  a volatile field, matching `AlbumArtCache`, `LanguageManager`,
  `ThemeManager`, `StorageAccessHelper`, `SongDatabase`, and
  `NotificationController`.
- `PlaylistManager.getInstance(Context)` — same shape.
- `ThemeHelper.getInstance(Context)` — same shape; the method-level
  `synchronized` and the inner `synchronized` block are both removed,
  keeping the double-checked pattern.
- `OverlayManager.sInstance` — `volatile` removed. `init(Context)` stays
  a `synchronized` method; `getInstance()` stays private.

**Names.** Five names were brought into line with the codebase's
one-name-per-concept rule.

- `ThemeHelper.mAppContext` → `mContext`. Every other class that holds
  an application context names the field `mContext`.
- `MetadataReader.closeQuietly` and `MetadataWriter.closeQuietly` →
  `closeSafely`. The `*Safely` suffix is the codebase's contract for
  "swallows the exception, the failure is not interesting".
- `EqualizerManager.getBassBoostStrength()` →
  `getAppliedBassBoostStrength()`; `getSurroundStrength()` →
  `getAppliedSurroundStrength()`. The two getters read the effect
  object's rounded value, not the persisted value; the new names make
  the distinction visible at the call site.
- `NavigationController.mCurrentState` → `mState`;
  `ToastManager.mCurrentState` → `mState`. The codebase uses `mState`
  for the primary state machine of a class that has one, and
  `m<Domain>State` for the second machine in a class that has two.
  Neither class has two.
- `MusicService.mMachineHandler` → `mMainHandler`. The handler is bound
  to the main looper; every other class names it `mMainHandler`.

**Removals.** Three pieces of code that were a second name for a fact
the codebase already names once.

- `EqualizerManager.getPitchLevel()` — deleted. The method was
  documented as "Equivalent to `getPitch()`" and had no callers.
- `NavigationController.getCurrentState()` — deleted. The method
  returned the internal state's name as a `String`; the state's name
  is already available through `getDebugInfo()`, which is the class's
  debug surface.
- `MusicService.mIsPlaying` — deleted. The field was written in eight
  methods and read in none. The authoritative answer to "is the
  service playing right now?" is the `isPlaying()` method, which
  derives from `mState`.

**Helpers.** Four fixes that bring a helper and its callers into the
shape the codebase's other helpers already take.

- `LanguageManager.shutdown()` — added. Every other singleton in the
  lazy-access group has one (`AlbumArtCache`, `OverlayManager`,
  `ToastManager`, `NavigationController`, `NotificationController`);
  `LanguageManager` was the one that was missing it. The new method
  clears the cached localized context before clearing the singleton
  reference.
- `PlaybackWindow.pathsEqual(String, String)` — deleted; its call
  sites in `equals(Object)` now use `Objects.equals(...)`. The
  method's body was the definition of `Objects.equals`.
- `MusicService.registerReceiverSafely(BroadcastReceiver, String)` and
  `MusicPlayer.registerReceiverSafely(BroadcastReceiver, String)` —
  the two helpers now take an `IntentFilter` rather than building one
  from a `String` action. Every call site builds the filter inline,
  so the filter's action list is visible at the call site.
  `PlaylistActivity.registerReceiverSafely(BroadcastReceiver,
  IntentFilter)` already had this signature; the other two classes now
  match it.
- `CreditsDialog.getLocaleContext()`, `LanguageDialog.getLocaleContext()`,
  `PresetDialog.getLocaleContext()`, and `ThemeDialog.getLocaleContext()`
  — each now logs a warning at `Log.w` before falling back to the
  activity context, matching what `EqualizerManager`,
  `NotificationController`, `PlaylistManager`, and
  `StorageAccessHelper` already do. This is the one place in this
  release where user-visible behavior changes: a failure to reach the
  language manager now leaves a warning in logcat.

### Under the Hood

- `EqualizerManager.java` — singleton shape; `getBassBoostStrength`
  and `getSurroundStrength` renamed; `getPitchLevel` deleted.
- `PlaylistManager.java` — singleton shape.
- `ThemeHelper.java` — singleton shape; `mAppContext` renamed to
  `mContext`.
- `OverlayManager.java` — `volatile` removed from `sInstance`.
- `LanguageManager.java` — `shutdown()` added.
- `MetadataReader.java`, `MetadataWriter.java` — `closeQuietly`
  renamed to `closeSafely`; call sites updated.
- `CreditsDialog.java`, `LanguageDialog.java`, `PresetDialog.java`,
  `ThemeDialog.java` — `TAG` constants added; `getLocaleContext()` now
  logs a warning on failure.
- `PlaybackWindow.java` — `pathsEqual` deleted; `equals` uses
  `Objects.equals`.
- `NavigationController.java` — `mCurrentState` renamed to `mState`;
  `getCurrentState()` deleted.
- `ToastManager.java` — `mCurrentState` renamed to `mState`.
- `MusicService.java` — `mMachineHandler` renamed to `mMainHandler`;
  `mIsPlaying` deleted; `registerReceiverSafely` now takes an
  `IntentFilter`.
- `MusicPlayer.java` — `registerReceiverSafely` now takes an
  `IntentFilter`.

---

## System Requirements

- **Minimum SDK:** Android 6.0 (API 23)
- **Target SDK:** Android 15 (API 35)
- **Recommended RAM:** 1 GB or higher

---

## Download

Download the APK from the Assets section of this release.

---

## Installation

1. Download the APK file
2. Enable "Unknown Sources" in your device settings
3. Open the APK file and tap "Install"

---

## First Time Setup

1. Tap the MANAGER button on the main screen
2. Tap IMPORT FOLDER to select your music folder
3. Tap UPDATE to save your selection
4. The app will scan your folder and build the playlist

---

## How to Use

| Feature | How to Access |
|---------|---------------|
| Equalizer | Tap EQUALIZER button on main screen |
| Metadata Editor | Tap METADATA button on main screen |
| Folder Manager | Tap MANAGER button on main screen |
| Playlist | Tap PLAYLIST button on main screen |
| Themes | Tap LLAMA button on main screen, then THEMES |
| Language | Tap LLAMA button on main screen, then the language button in the version header |
| Volume gesture | Swipe up or down along the album-art edge |
| Previous / next song | Swipe left or right across the album art, or tap the transport buttons |
| Playlist fast-scroll | Drag vertically along the left or right edge of the playlist |
| Sort mode | Tap SORT in the playlist header |

---

## Credits

- jaudiotagger by jthink Ltd
- FontAwesome by Fonticons, Inc.

---

## License

Licensed under the MIT License

Copyright (c) 2026 Richard Korbla Adzido

Made with love in Ghana