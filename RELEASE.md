# Llama Music Player v1.6.1

**Release date:** November 21, 2026

Version 1.6.1 does one thing: a cold-start restore of a paused
session no longer auto-plays when a second `setPlaylist`
re-prepares the song.

Llama Music Player itself remains licensed under the MIT License.

For the full change history, see the `[1.6.1]` entry in the
[CHANGELOG](CHANGELOG.md). For technical specifications, installation,
and usage, see the [README](README.md).

---

## What's in this release

### Fixed: paused cold-start restore stays paused across re-prepares

When the app was paused before it exited, a cold start restores
the playlist and holds the song paused. That decision lived only
in `mPauseAfterPrepare`, which `onPrepared()` consumes, so a
second `setPlaylist` that changed the target — the library load
completing, a sort change — re-prepared the song with auto-start
re-armed and it started playing on its own.

The cold-start branch now also records the decision in
`mStickyPause`, and `beginTransition()` re-arms the pause from
it on every prepare, so the re-prepared song still lands
paused. An explicit play action clears the hold: tapping a
song, Next, Previous, or the play button, and a real
transition to PLAYING. A playing restore is unaffected: it
was never held, and it still resumes.

### Existing behavior preserved

No change to the 1.6.0 fix: the playback scope dialog still
opens on the current scope. No change to the 1.5.9 fixes: the
cold-start auto-start commit, the narrow-landscape docking
gate and seek-bar hide, the artist-group centering, or the
seek-bar drawable rename. No change to scope matching,
repeat, shuffle, sort, search filtering, favorite persistence,
or click debounce. No change to ProGuard rules.