# Llama Music Player v1.4.2

**Release date:** November 7, 2026

Version 1.4.2 refines the playback-scope dialog so an empty filter
selection is presented and applied for what it is. For the full change
history of this release, see the `[1.4.2]` entry in the
[CHANGELOG](CHANGELOG.md). For technical specifications, installation,
and usage, see the [README](README.md).

---

## What's in this release

### Playback scope

A filter mode with no values checked is a scope that matches nothing.
The dialog now shows that selection as it is: the mode stays selected,
its filter list stays expanded, and the header count reads zero. The
selection is no longer rewritten into All Songs; that scope is reached
through the All Songs row. APPLY is enabled when the selection differs
from the scope the dialog opened with, so an empty selection from a
different starting scope can be committed and re-applying the current
scope cannot. A preference change made after APPLY is tapped cancels
the queued commit and restores the button to the state the current
selection warrants.

An empty filter scope reaches the service like any other scope. The
service builds an empty scoped playlist from it and leaves the playlist
view empty until the user changes the scope; the scope is not rewritten
into All Songs at the service boundary.