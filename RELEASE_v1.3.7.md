# Llama Music Player v1.3.7

**Release date:** Oct 25, 2026

### Overview

This release ships fixes to the incoming-file pathway, the
notification's play/pause icon, the transport control contrast, the
Credits license viewer, and four smaller correctness issues in the
activity. It also adds two colors to the theme list.

An incoming audio file is an explicit play request. The activity hands
the file's path to the bound service through a direct main-thread
method call. The service stores the path as its newest incoming-file
request and, when the playlist already contains it, asks the state
machine to transition to it on the same turn. When the playlist does
not yet contain the path, the path stays parked and the reconcile that
follows the insert's rebuild consumes it. The slot is cleared only
when the transition it names actually begins, so a request that
arrives while another transition is preparing survives the prepare and
plays after it settles. No broadcast is involved, so there is no
delivery window between the activity and the service.

The notification's play/pause icon reads the destination of the
current transition. The destination is derived from `isPlaying()`,
which accounts for both the auto-start flag and the pending-pause
flag, so the value the icon reports and the value the state machine
settles on are the same. Every state change that shifts
`isPlaying()` during PREPARING pokes the controller on the same
main-thread turn, and the audio-focus gain path handles PREPARING
and READY.

The transport controls draw their inactive stroke and text in a
value that holds visible contrast against both the app's `#121212`
background and the palette's lightest theme color, Pure White at
`#FFFFFF`. The Credits dialog's license viewer cross-fades only when
the requested license differs from the one on screen. The theme list
gains two colors at its end, under a category the source labels
Muted tones: Pure White and Frosted Glass.

This release also ships four smaller correctness fixes in the
activity. A cancelled album-art swipe now returns the carousel to
rest instead of committing a gesture the user did not complete. A
content-URI file whose MediaStore query returns no rows now falls
back to the copy path instead of being dropped silently. Two callers
on different threads can no longer both claim the update slot in
`safeUpdateUI`. A failed loose-track insert now clears the service's
pending path, so a later reconcile cannot resolve to a path that was
never added to the playlist.

### Fixed

- **Incoming file plays on every request** — When a user opens an
  audio file from another app, the activity hands the file's path to
  the bound service through a direct main-thread method call rather
  than a broadcast. The service stores the path as the newest
  incoming-file request and, when the playlist already contains it,
  asks the state machine to transition to it on the same turn. When
  the playlist does not yet contain the path, the path stays parked
  and the reconcile that follows the insert's rebuild resolves it.
  The slot is cleared only when the transition it names actually
  begins, so a request that arrives while another transition is
  preparing survives the prepare and plays after it settles. The
  prepare-completion callback re-checks the pending slot before
  settling, so a request whose path landed during the prepare still
  re-enters reconciliation even when the target and current indices
  agree. No broadcast is involved, so there is no delivery window
  between the activity and the service.

- **Notification play/pause icon never drifts from the state
  machine** — The icon reads `isPlaying()`, which reports the
  destination of a PREPARING transition from both the auto-start
  flag and the pending-pause flag. A pause scheduled during a
  prepare — a notification pause tap, a media-session pause, a
  hardware pause key, an audio-focus loss — sets
  `mPauseAfterPrepare`, and the icon flips at the moment the
  request is accepted rather than when the prepare completes. The
  same holds in reverse: a resume requested during a prepare
  clears the pending pause and re-arms the auto-start, and the
  icon follows. Every state change that shifts `isPlaying()`
  during PREPARING pokes the notification controller on the same
  main-thread turn.

- **Audio focus gain handles PREPARING and READY** —
  `AUDIOFOCUS_GAIN` resumes from PAUSED, PREPARING, and READY.
  When the state is PREPARING, the gain branch clears the pending
  pause and re-arms the auto-start; when the state is READY and
  the prepare completed while the loss was active, the gain
  branch starts playback. Every branch of the focus handler pokes
  the controller, so a focus event that does not change the state
  — a duck, a gain without a remembered pause, a loss while
  already paused — does not leave the icon from the previous
  transition.

- **Control button contrast against the lightest theme** — The
  transport buttons draw their inactive stroke and text in a fixed
  grey. That value is read against both the app's `#121212`
  background and the palette's lightest theme color, Pure White at
  `#FFFFFF`, and a single grey cannot hold its edge against both
  ends. The inactive color is now `#A0A0A0`, carried as
  `ThemeHelper.CONTROL_BUTTON_INACTIVE_COLOR`. Every other surface
  that reads `ThemeHelper.COLOR_TEXT_PRIMARY` keeps its current
  value.

- **Credits license viewer skips the cross-fade when the tapped
  license is already on screen** — The viewer cross-faded even when
  the tapped license was the one displayed. The touch feedback still
  plays on every tap; the cross-fade now runs only when the requested
  license differs from the one on screen.

- **Cancelled album-art swipe returns the carousel to rest** — When
  the framework cancels a touch sequence mid-carousel — a parent
  view takes the gesture, the window loses focus, a system
  interruption arrives — the activity released the carousel with
  the cancel event's position interpreted as a horizontal
  displacement. The strip could commit a swipe the user never
  meant to make, dispatching a next or previous request. The
  cancel branch now releases the carousel with a displacement of
  zero, so the strip returns to rest and no transport request is
  dispatched. Only an `ACTION_UP` reaches `endSwipe` with a
  meaningful displacement.

- **Content-URI file resolution falls back to copy when MediaStore
  has no DATA column** — `getFilePathFromUri` previously reached
  its copy-to-temp-file fallback only when the MediaStore query
  raised. A provider that returned an empty cursor — no `DATA`
  column exposed, or no rows for the given URI — left the resolved
  path null and the file was dropped silently. The fallback is now
  reached from both cases: an exception is logged and the copy is
  attempted, and an empty cursor proceeds to the copy without
  logging. Files from providers that do not implement the media
  store schema play through the same path as any other incoming
  file.

- **Loose-track insert failure clears the service's pending path**
  — When the insert of an incoming loose track fails, the activity
  now calls `MusicService.clearPendingSongPath()` on the bound
  service in addition to clearing its own `mPendingIncomingPath`
  field. Without the service-side clear, a subsequent reconcile
  could resolve the still-parked path to an index the rebuild had
  already rejected, and start a transition to a song that was
  never added to the playlist.

### Added

- **Pure White theme color** — A pure white at index 30 of
  `ThemeManager.COLOR_VALUES`. The hex value is `#FFFFFF`. The
  theme's display name is carried in `R.array.theme_names` in
  each locale. Against the player's `#121212` background, Pure
  White is the lightest and quietest of the muted entries: it
  reads as a clean paper surface, so it works as the palette's
  highest-contrast neutral when the user wants one.

- **Frosted Glass theme color** — A diffuse cool blue-green at
  index 31 of `ThemeManager.COLOR_VALUES`. The hex value is
  `#9DBCBC`. The theme's display name is carried in
  `R.array.theme_names` in each locale. The value is chosen for
  the player's `#121212` background: the swatch reads as
  illuminated translucency against near-black, not as a solid
  panel, and its cool cast supplies the glass read the swatch
  itself cannot carry. `#9DBCBC` sits below the Pure White swatch
  in value, so the two muted entries occupy distinct ends of the
  palette without crowding each other.

- **Both new colors resolve in every locale** — Each locale's
  `theme_names` string-array carries both entries, so the array's
  length matches `COLOR_VALUES` at 32 and
  `ThemeManager.getColorName(int)` resolves a name for both new
  indices in every language.

### Under the Hood

- `MusicService.isPlaying()` — the PREPARING arm returns
  `mAutoStartOnPrepared && !mPauseAfterPrepare`.

- `MusicService.pokeNotificationIfAttached()` — a new helper that
  pokes the controller's awareness poll when one is installed and
  is a no-op when it is not. Called from every state change that
  shifts `isPlaying()`.

- `MusicService.beginTransition(int, int)` — pokes the controller
  after setting the state to PREPARING, so the icon flips on the
  turn the transition begins. Consumes the pending incoming-file
  path when it names the target, at the moment the transition
  begins rather than at reconcile time; a pending path that names
  a different target survives and is re-checked on the completion
  of this transition.

- `MusicService.togglePlayPause()` — the PREPARING arm reads the
  current destination and toggles it. A transition heading toward
  playback gets a pending pause and a cleared transient-loss
  flag; a transition already headed toward pause has the pending
  pause cleared and the auto-start re-armed. Both branches poke
  the controller.

- `MusicService.pause()` — the PREPARING arm pokes the controller
  after setting `mPauseAfterPrepare`.

- `MusicService.resume()` — gained a PREPARING arm that clears
  `mPauseAfterPrepare` and re-arms `mAutoStartOnPrepared` when
  either was set, and pokes the controller.

- `MusicService.handleAudioFocusChange(int)` — the
  `AUDIOFOCUS_GAIN` branch handles PAUSED, PREPARING, and READY.
  The PREPARING case clears the pending pause and re-arms the
  auto-start; the READY case runs the READY-to-PLAYING
  transition. Every branch pokes the controller.

- `MusicService.setupMediaSession()` — the `onPlay` and `onPause`
  callbacks gained PREPARING arms that reflect the request and
  poke the controller.

- `MusicService.handleMediaButton(int)` — the
  `KEYCODE_MEDIA_PLAY` and `KEYCODE_MEDIA_PAUSE` arms gained
  PREPARING handling and poke the controller.

- `MusicService.reconcile()` — resolves a pending incoming-file
  path against the current playlist before the need check. A
  pending path that resolves to an index overrides the
  reconcile's ordinary target; one that resolves to the current
  song is consumed without a transition. The no-op branch clears
  `mDeferredReconcile` before returning, so a request that
  resolved to the current song cannot be re-entered by a later
  completion callback.

- `MusicService.setPlaylist(ArrayList<Song>, boolean)` — the
  PREPARING branch re-resolves the target's index against the
  new playlist, then calls `reconcile()` before broadcasting.
  The pending slot is not consumed here; it is consumed only
  inside `beginTransition` when the transition it names actually
  begins, so a request that arrives during the prepare survives
  the rebuild.

- `MusicService.requestIncomingFile(String)` — a new method
  that records the given path as the newest incoming-file
  request and re-enters reconciliation. Called from the bound
  service on the main thread by the activity's incoming-file
  handoff. The newest request overwrites the field — the user
  who opens a second file while the first is still preparing
  gets the second file, not the first.

- `MusicService.mPendingSongPath` — the incoming-file request
  slot. Set by `requestIncomingFile` and by `setPendingSongPath`,
  read by `reconcile` and by the prepare-completion callbacks
  (`onPrepared` and `onPlaybackStarted`). Cleared only when
  `beginTransition` begins the transition the path names, or when
  `reconcile` resolves it to the current song. A rebuild that
  runs before either of those leaves the slot set, so a request
  that arrives during a prepare survives the prepare.

- `MusicPlayer.handOffIncomingPath(String)` — hands the path to
  the bound service through a direct call when the service is
  bound, or parks it on `mPendingIncomingPath` when the bind has
  not yet completed. The parked path is handed off from
  `onServiceConnected`. No broadcast is involved on either path.

- `MusicPlayer.insertLooseTrackAndHandOff` — the `onError`
  branch now calls `MusicService.clearPendingSongPath()` when
  the service is bound. The activity's own parked field is
  cleared first, as before.

- `MusicPlayer.safeUpdateUI(Runnable)` — the update slot is
  claimed with `mIsUpdatingUI.compareAndSet(false, true)`
  instead of a read-then-write. Two callers that both observed
  `false` can no longer both post the update. `mPendingUIUpdate`
  is now `volatile`, so the queued update written by a losing
  caller is visible to the main thread's `finally` block when
  it decides whether to re-enter.

- `MusicPlayer.handleGestureUp(View, MotionEvent)` — an
  `ACTION_CANCEL` passes a displacement of `0f` to
  `endSwipe`. Only an `ACTION_UP` reaches `endSwipe` with the
  event's actual horizontal displacement.

- `MusicPlayer.getFilePathFromUri(Uri)` — the copy-to-temp-file
  fallback is reached from both the exception case and the
  empty-cursor case. A provider that returns no rows for a valid
  content URI no longer leaves the path null.

- `ThemeManager.COLOR_VALUES` — two entries appended at indices
  30 and 31, under a `// Muted tones.` category comment. Index 30
  carries `#FFFFFF` under the name Pure White. Index 31 carries
  `#9DBCBC` under the name Frosted Glass.

- `ThemeHelper.CONTROL_BUTTON_INACTIVE_COLOR` — a new constant at
  `0xFFA0A0A0`, used by `updateControlButton(...)` for the
  inactive stroke and inactive text of the transport buttons. It
  is separate from `COLOR_TEXT_PRIMARY`, which keeps its value and
  continues to serve every other surface that reads it.

- `CreditsDialog.updateLicenseViewer(String)` — returns early when
  the requested text equals `mCurrentLicenseText`, which is set
  before the cross-fade begins so a second tap during a running
  fade is also a no-op.

- `res/values/strings.xml` and every locale's `strings.xml` —
  the `theme_names` string-array carries two additional entries
  in each file, so the array's length matches `COLOR_VALUES` and
  `ThemeManager.getColorName(int)` resolves a name for both new
  indices in every locale.

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