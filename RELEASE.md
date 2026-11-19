# Llama Music Player v1.5.4

**Release date:** November 19, 2026

Version 1.5.4 improves playback-scope matching across Android storage
path representations and corrects metadata refresh propagation across
the service, player screen, and playlist display.

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

### Player and playlist displays refresh narrowly

When the edited song is currently playing, the player screen refreshes
its displayed title and artist from the service's refreshed song state.
The album-art carousel cache is invalidated only when artwork changes or
is removed. The playlist replaces only the affected display row, notifies
that row, and guards the playing-index update against a missing row.

### Existing playback behavior is preserved

The changes do not introduce a new playback queue or transition owner.
The service remains authoritative for playback state, and existing queue
ordering, transition behavior, repeat and shuffle behavior, and favorite
persistence remain outside the metadata refresh path.
