# Llama Music Player v1.6.0

**Release date:** November 21, 2026

Version 1.6.0 does one thing: the playback scope dialog now
opens on the current scope instead of at the top.

Llama Music Player itself remains licensed under the MIT License.

For the full change history, see the `[1.6.0]` entry in the
[CHANGELOG](CHANGELOG.md). For technical specifications, installation,
and usage, see the [README](README.md).

---

## What's in this release

### Fixed: playback scope dialog opens on the current scope

The dialog opened at the top, so a current scope low in
the list — a late mode such as Albums, or a checked filter
deep in a long genre, artist or album list — was off-screen
and the user had to hunt for it.

On open the dialog now scrolls to the current scope when
it is not already in view: the first checked filter of the
current kind, or the kind's mode row when it has no checked
values. The scroll runs twice, on purpose: once at open,
which covers folders and the mode rows, and again once the
genre, artist and album lists finish loading, which refines
it to the checked filter. A target already in view never
scrolls, and the scroll is an instant jump, not an
animation, so the dialog opens on the selection.

### Existing behavior preserved

No change to the 1.5.9 fixes: the cold-start auto-start
commit, the narrow-landscape docking gate and seek-bar
hide, the artist-group centering, or the seek-bar drawable
rename. No change to scope matching, repeat, shuffle, sort,
search filtering, favorite persistence, or click debounce.
No change to ProGuard rules: the 1.6.0 change adds no new
subscriber and no new kept field.