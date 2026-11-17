# Llama Music Player v1.5.2

**Release date:** November 17, 2026

Version 1.5.2 fixes the Credits dialog's theme and language buttons
becoming unusable after the user reselected the current item. The
pickers' selection callbacks replaced the dismiss callbacks that
cleared the host dialog's picker-open flags. The pickers now notify
the host through a separate dismiss callback, preserving selection
handling and allowing the buttons to open their pickers again.
For the full change history, see the `[1.5.2]` entry in the
[CHANGELOG](CHANGELOG.md). For technical specifications,
installation, and usage, see the [README](README.md).

---

## What's in this release

### The theme and language buttons stay usable after reselection

The Credits dialog tracks whether each picker is showing so a
second tap cannot open another copy. The theme picker and the
language picker each install a dismiss listener to report the
selection. Because a `Dialog` only stores one dismiss listener, the
selection callback replaced the callback that cleared the host's
picker-showing flags. Reselecting the current item therefore
returned without changing the theme or language and left the guard
in place; later taps on the same button were ignored.

The fix adds a separate dismiss callback on both pickers, so the
host clears its guard on every dismissal while the picker continues
to fire its selection callback. Selecting the current theme or
language still closes the picker without broadcasting or persisting
a new value, but the button is no longer left unusable after that
close.

### The selection lifecycle stays unchanged

The pickers still fire their selection callbacks from the dialog's
own dismiss flow, after the dialog window has detached. The host
also receives its separate dismissal callback on that lifecycle
turn, clearing the button's guard whenever the picker closes.