# Llama Music Player v1.3.7

**Release date:** Oct 25, 2026

### Overview

Five changes ship in this release: the incoming-file pathway plays
every request, the notification's play/pause icon never drifts from
the state machine's destination, the transport controls hold their
contrast against the lightest theme, the Credits dialog skips a
cross-fade that has nothing to change, and the theme list carries two
new colors.

An incoming audio file is an explicit play request. The activity
parks the file's path in the service, the service resolves the path
as the anchor of the playlist rebuild the insert triggers, and the
state machine transitions to it. The path is preserved through the
rebuild's cold-start branch, and also through the branch that runs
while a transition is already in flight. The pending path overrides
the in-flight target; the transition to it begins once the current
prepare settles.

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

### Fixed

- **Incoming file plays when a transition is already in flight** —
  When a user opens an audio file from another app while a next or
  previous request is preparing, the file's path is parked in the
  service as a pending path, and the playlist rebuild that follows
  the insert resolves the path as the anchor of the next
  transition. The rebuild has two branches that preserve the
  target through an in-flight transition. The cold-start branch
  carries the pending path through unchanged and starts the
  transition when the rebuild returns. The in-flight branch
  recognises the pending path and overrides the target with it.
  The reconcile that follows sees PREPARING and sets
  `mDeferredReconcile`, and the prepare completion callback
  re-enters reconciliation with the newest target. The new song
  plays through the same transition path a next or previous
  request uses.

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
  turn the transition begins.

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

- `MusicService.reconcile()` — the no-op branch clears
  `mDeferredReconcile` before returning, so a request that
  resolved to the current song cannot be re-entered by a later
  completion callback.

- `MusicService.setPlaylist(ArrayList<Song>, boolean)` — the
  PREPARING branch recognises the pending path and overrides the
  target with it before calling `reconcile()`. The pending slot
  is consumed only when the path it names was found in the
  rebuilt playlist.

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