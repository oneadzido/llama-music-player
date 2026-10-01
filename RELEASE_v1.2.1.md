# Llama Music Player v1.2.1

**Release date:** Oct 1, 2026

## Overview

The album-art carousel now animates on every kind of song transition —
a track ending on its own, a headset media button, a shuffle pick, a
repeat-one restart — with the same visuals a swipe or a button press
produces. Repeat-one has a dedicated animation that matches its
behaviour: a small glide toward the neighbour and back, whether the
restart came from the user's press or the track looping on its own.
The playlist sort has been moved off the main thread in both the
service and the adapter, so the sort overlay's spinner rotates freely
throughout the operation, and the overlay now waits for the
RecyclerView to settle with the new ordering before it dismisses, so
its exit reveals a list already in its final state.

## What's New

- **Carousel animates on automatic transitions** — A song ending on
  its own, a headset media button skip, a shuffle pick, or a
  repeat-one restart all now animate the album-art strip exactly as a
  transport button press does. Previously these transitions arrived as
  instantaneous snaps: the audio changed, the strip's slots updated,
  but nothing moved. The animation is now the default, and only a
  re-publication of the same window — a pause, a resume, a seek, a
  metadata poke — leaves the strip still.

- **Repeat-one has its own animation** — Under repeat-one, a next or
  previous press restarts the current song rather than advancing. The
  strip's response matches that behaviour: instead of the full commit
  slide a normal advance produces, it runs a below-threshold nudge — a
  small glide toward the neighbour position and back to rest. The
  animation reads as an acknowledgement that the song restarted, not
  as the beginning of an advance.

- **Rejected button presses glide back** — Previously, under
  repeat-one, a next or previous press started the full commit slide
  toward the neighbour and then jumped straight back to rest when the
  service refused to advance. The return is now a glide using the same
  method a below-threshold swipe uses, so the press and the release of
  a committed swipe under the same mode produce the same visible
  shape.

- **Sort reveals a settled list** — The sort overlay no longer hides
  on a timer measured from the sort's start. It hides only after the
  RecyclerView has laid out and drawn the frame containing the sorted
  list, so the moment the overlay disappears is the moment the user
  sees the final ordering. Nothing repaints after the overlay's exit.

- **Sort no longer stalls the spinner** — The playlist reorder ran in
  two places on the main thread. The service's sort-mode receiver
  sorted the authoritative playlist inline in its broadcast callback,
  and the playlist adapter ran `DiffUtil` on the fully reordered list
  when the activity handed it back. Both are now off the main thread.
  The service sorts on a dedicated executor, and the adapter is told
  through a new overload that the update is a pure reorder, skipping
  the `DiffUtil` pass whose Myers algorithm degenerates to O(n²) on a
  full reorder.

## Fixed

- **Shuffle auto-advance disagreed with the button and the window** —
  Under shuffle with repeat off, the automatic advance at the end of a
  track used the positional next index, while the transport button and
  the carousel's window both used the shuffle-chosen pick. The
  carousel therefore snapped without animating on every natural track
  end under that mode combination, because the incoming current path
  matched neither of the previous window's neighbours. The
  auto-advance now uses the same picker the button and the window use,
  so all three agree on the destination.

- **Repeat-one auto-loop did not return** — Under repeat-one, when a
  track looped on its own, the strip used to slide the neighbour fully
  into the viewport and then cut back to the current song. The visual
  suggested an advance had happened and been reversed. The animation
  is now the below-threshold nudge, matching what a press under the
  same mode produces.

- **Repeat-one next/prev button had no matching animation** — The
  window used to publish null neighbours under repeat-one, so the
  carousel had nowhere to slide. A next or previous press showed no
  animation while the audio restarted. The window now publishes the
  positional neighbours — the songs a next or previous request would
  advance to if repeat-one were off — and the carousel slides toward
  one of them before returning.

## Under the Hood

- `PlaybackWindow` gained a `generation` field: the service's
  monotonic transition counter at the moment the window was produced.
  Equality and hash code now include it, so two windows with the same
  three paths but different generations are distinct values. A
  re-publication (a pause, a resume, a seek, a metadata poke) carries
  the same generation; a fresh transition (a prepare for a new song, a
  reprepare that restarts the current song, a transition to idle)
  advances it. The carousel uses this to tell a restart apart from a
  re-publication even though the three paths are unchanged in both
  cases.

- `MusicService.peekNextIndex(int)` and `peekPreviousIndex(int)` no
  longer special-case repeat-one. They return the positional or
  shuffle-chosen neighbour under every repeat mode. The window
  therefore always has somewhere for the carousel to slide toward.

- `MusicService.getWindowSnapshot()` publishes the peek result
  without nulling the neighbours for repeat-one, and carries
  `mGeneration` on the returned window.

- `MusicService.playNextSong()` and `playPreviousSong()` restart the
  current song under repeat-one by setting `mTargetIndex = baseIndex`
  and `mForceReprepare = true`. The reprepare is what makes
  `reconcile()` begin a fresh transition even though the target index
  has not changed. The three-way switch semantics — off, all, one —
  are now uniform across every entry point: transport buttons, headset
  media buttons, notification actions, and the automatic advance at
  the end of a track.

- `MusicService.onCompletion()` uses `computeNext` whenever shuffle is
  on, under repeat off or all. Previously the repeat-off shuffle path
  was positional, which made the auto-advance disagree with both the
  window and the button.

- `MusicService` gained a single-thread `mSortExecutor`. The
  sort-mode receiver captures the current playlist, the anchor path,
  and the requested sort mode on the main thread, sorts a copy on the
  executor, and posts the swap and the broadcast back to the main
  thread. A newer sort-mode change supersedes an older one through a
  comparison of the captured mode against the current field; a
  superseded task exits without touching state. The executor is shut
  down in `onDestroy`.

- `MusicPlayer.mUpdatePlayerReceiver` now passes the broadcast's
  `generation` extra to the `PlaybackWindow` constructor as its fourth
  argument. The generation was already parsed for the activity's own
  gate; the change is one line.

- `MusicPlayer.playNextSong()` and `playPreviousSong()` choose the
  carousel animation from the service's current repeat mode: the
  below-threshold nudge when repeat-one, the full commit slide
  otherwise. The mode check is a single `getRepeatMode() == 2` and
  does not race in practice — repeat mode is a user setting that
  changes far slower than a button press round-trip.

- `AlbumArtCarousel` gained a `SwipeState.RESTARTING` state for the
  repeat-one nudge. Both legs of the animation run inside this state
  so a re-publication during the animation is absorbed without
  interrupting it, and only a newer generation aborts it. The state is
  also entered by the automatic restart when the incoming window
  carries the current path unchanged but an advanced generation.

- `AlbumArtCarousel` gained a `RESTART_NUDGE_FRACTION = 0.25f`
  constant, separate from `COMMIT_THRESHOLD_FRACTION = 0.50f`. The
  nudge is deliberately half the commit threshold's travel: it reads
  as a gentle peek rather than a full attempt. The two constants are
  independent so the commit threshold can be tuned without affecting
  the nudge and vice versa.

- `AlbumArtCarousel.setWindow`'s COMMITTING mismatch branch now calls
  `animateToRest` instead of `snapToRest`. The strip glides back from
  wherever the animation had reached rather than jumping, which is
  what produces the rejected-press glide. The same `animateToRest`
  is used by every other return path — below-threshold swipe
  spring-back, refused committed swipe, refused button press, snapshot
  timeout, and the return leg of the nudge — so the return animation
  is guaranteed identical across all cases.

- `AlbumArtCarousel.adoptInIdle` now takes the previous generation and
  recognises two forms of transition: a song change (the current path
  changed to a neighbour — full commit slide) and a restart (the
  current path is unchanged but the generation advanced —
  below-threshold nudge). The generation is the only signal that
  distinguishes them, because the three paths are identical in both
  cases.

- `AlbumArtCarousel.nudgeToNext()` and `nudgeToPrevious()` are the
  public methods the activity uses to run the repeat-one nudge. They
  have the same idle/viewport/neighbour guards as `animateToNext()` /
  `animateToPrevious()` and are the nudge equivalents of those commit
  methods. The carousel has no knowledge of repeat modes; the caller
  selects the animation.

- `AlbumArtCarousel.calculateSampleSize` parameter names were expanded
  from `reqWidth` / `reqHeight` to `requestedWidth` / `requestedHeight`
  to match the codebase's convention of full names for variables.

- `PlaylistActivity` gained a `mSortGeneration` field advanced by
  every `cycleSortMode` call. The hide callback that runs after the
  RecyclerView has settled checks it against the generation its cycle
  was started with, so a superseded cycle cannot take the overlay down
  on the current cycle's behalf.

- `PlaylistActivity.cycleSortMode()` now updates the adapter and
  performs the focus scroll synchronously in the same main-thread
  turn, via a new `scrollToCurrentSongImmediately()` method. The new
  order and the new scroll position land in the same RecyclerView
  layout pass. The overlay is hidden only after that settled frame has
  been drawn, via a new `hideSortOverlayAfterListSettles()` method
  that registers a `ViewTreeObserver.OnPreDrawListener`, removes it on
  the first callback, and posts the hide from inside it. The
  `SORT_OVERLAY_MIN_DURATION_MS` floor is applied on top of the settle
  wait by `scheduleSortOverlayHideAtMinDuration`.

- `PlaylistAdapter.updateList(List<Song>)` gained an overload taking a
  `reorderOnly` flag. The original signature is preserved and
  delegates with the flag set to `false`. A caller that knows the two
  lists contain the same songs in a different order — the sort cycle
  is the case this was added for — passes `true` and the adapter
  replaces the whole list in one step instead of running `DiffUtil`.
  The flag is a hint, not a contract: a caller that is uncertain can
  pass `false` and let `shouldBulkReplace` decide.

- `PlaylistActivity.cycleSortMode()` calls `updateList` with
  `reorderOnly = true`, opting in to the fast path. Every other
  `updateList` call site in the file is unchanged and keeps the
  `shouldBulkReplace` / `DiffUtil` behaviour for the small updates the
  app already uses it for — genre updates, metadata edits, incremental
  progress additions.

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

1. Tap the MANAGER button in the top right corner
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