# Llama Music Player v1.5.4

**Release date:** November 19, 2026

Version 1.5.4 improves playback-scope matching across Android storage
path representations, enforces the selected scope across all playback
navigation and recovery paths, and corrects metadata refresh propagation
across the service, player screen, and playlist display.

Playback-scope comparisons normalize path separators, redundant path
segments, and common aliases for primary shared storage before matching
loose tracks and selected folders. Folder matching retains path-boundary
checks so a selected folder does not accidentally include a similarly
named sibling folder.

Metadata refresh preserves unchanged raw metadata values, supports
explicitly cleared text fields, and retains unrelated song state. Artwork
caches are invalidated only when artwork changes or is removed. The
player screen refreshes the current song's title and artist, and the
playlist updates only the affected row and its playing indicator.

For the full change history, see the `[1.5.4]` entry in the
[CHANGELOG](CHANGELOG.md). For technical specifications, installation,
and usage, see the [README](README.md).

---

## What's in this release

### Playback scope matches equivalent storage paths

Loose-track and selected-folder comparisons now use a shared lexical path
normalizer. It standardizes path separators, removes redundant `.` and
`..` segments, and maps common primary shared-storage aliases such as
`/sdcard`, `/mnt/sdcard`, `/storage/self/primary`, and
`/storage/emulated/legacy` to `/storage/emulated/0`.

Folder matching still requires a path boundary after the selected folder,
so a selection such as `Music` does not include a sibling directory named
`Music2`. The normalization does not resolve symbolic links or change
path case.

Manual Next and Previous, automatic completion, prepare-failure recovery,
player-error recovery, and playback-stall recovery now choose destinations
from the active scoped playlist. Shuffle previews and shuffle history use
the same scoped candidate set. When a scope change excludes the current song,
the service transitions to an eligible song if one remains. If the scope has
no eligible songs, playback is stopped and released, current and pending
song state is cleared, and the player enters a genuine no-song state. An
excluded song cannot continue playing beneath placeholder metadata. An
in-flight prepare targeting an excluded song is invalidated before it can
commit. Favorite changes that remove the current song from a Favorites
scope follow the same reconciliation.

### Metadata refresh preserves the intended field state

Metadata fields are applied independently. A `null` field means no
change, an empty string means explicitly clear the field, and a non-empty
string replaces the stored value. Unchanged values are taken from raw
song metadata rather than localized display placeholders.

The service updates matching entries in its full and current playlists
without replacing unrelated song state. Favorite state, duration, and
existing URI values are retained unless an applicable update supplies a
replacement. Artwork is left alone when no artwork update is supplied;
removed or replacement artwork triggers the corresponding cache update.

### Playback navigation remains inside the selected scope

The service maps scoped songs back to the indices used by its existing
state machine. This keeps transition ownership and the current/pending
song grace period intact while preventing that grace period from leaking
into Next, Previous, shuffle, carousel-neighbour previews, or recovery
selection. Empty scopes have no navigation destination. Repeat-one keeps
its existing behavior for an in-scope current song, but does not pin
playback to a song excluded by a newly applied scope.

When a scope contains no eligible songs, the service now enters a true
no-song state: it stops and releases the player, clears current and pending
indices and the transitional playlist, clears playback artwork and
notification state, and publishes the empty scope. The UI therefore cannot
show placeholder title/artist text while an excluded song continues playing
underneath. If eligible songs remain, an excluded current song is replaced
by a valid scoped song. An out-of-scope in-flight prepare is invalidated
before it can commit. Favorite changes that remove the current song from a
Favorites scope follow the same rule. Scope publication generations are
advanced so delayed updates from an older scope cannot overwrite the latest
player display.

### Player and playlist displays refresh narrowly

When the edited song is currently playing, the player screen refreshes
its displayed title and artist from the service's refreshed song state.
The album-art carousel cache is invalidated only when artwork changes or
is removed. The playlist replaces only the affected display row, notifies
that row, and guards the playing-index update against a missing row.

### Existing playback behavior is preserved

The changes do not introduce a new playback queue or transition owner.
The service remains authoritative for playback state. The existing queue
and grace-period model remain in place; only candidate selection is now
consistently scoped. Repeat, shuffle, favorite persistence, and normal
background playback remain supported.
