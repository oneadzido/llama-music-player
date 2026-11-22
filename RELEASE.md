# Llama Music Player v1.6.1

**Release date:** November 22, 2026

Version 1.6.1 does two things, in this order. First, it
restores the v1.5.6 `MusicService`, because of the auto
playback resume bug: it surfaced in v1.5.7 and was noticed
in 1.6.0, while focus was on new features. Then, it fixes
a separate concern: the notification's Play button works
after a sticky restart, without opening the player.

Llama Music Player itself remains licensed under the MIT License.

For the full change history, see the `[1.6.1]` entry in the
[CHANGELOG](CHANGELOG.md). For technical specifications, installation,
and usage, see the [README](README.md).

---

## What's in this release

### Fixed: auto playback resume bug

The v1.5.6 `MusicService` is restored as the base. The auto
playback resume bug surfaced in v1.5.7 and was noticed in
1.6.0, while focus was on new features. Playback no longer
auto-resumes when it should stay paused.

### Fixed: notification Play works after a sticky restart

After the system killed the process, the sticky restart rebuilt
only the notification: it committed a paused snapshot of the last
song, but never rebuilt the playlist behind it. The service stayed
at `IDLE` with an empty playlist, so the notification's Play, Next,
and Previous buttons did nothing until the player was opened and
the activity called `setPlaylist`.

The restore now re-seeds the playlist behind the notification. It
loads the full library first, then makes exactly one playlist
call: the full library when it contains the last song, just that
song when the load fails or does not contain it. It prepares the
last song to `READY`, without starting playback, so Play resumes
the last song at its saved position and Next and Previous walk
the full library. Playback still does not start on its own, even
when the app was playing before it was killed: the paused prepare
is forced, and the user's Play tap is what resumes it. If
the restore lands first, opening the player attaches to the paused
restore instead of auto-resuming, and Play is one tap. A duplicate
restart does not re-seed or overwrite a live notification, and a
last song whose file is gone is not prepared.

### Existing behavior preserved

No change to the 1.6.0 fix: the playback scope dialog still
opens on the current scope. No change to the 1.5.9 fixes: the
cold-start auto-start commit, the narrow-landscape docking gate
and seek-bar hide, the artist-group centering, or the seek-bar
drawable rename. No change to scope matching, repeat, shuffle,
sort, search filtering, favorite persistence, or click debounce.
No change to ProGuard rules: the 1.6.1 change adds no new
subscriber and no new kept field.