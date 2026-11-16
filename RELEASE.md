# Llama Music Player v1.5.0

**Release date:** November 16, 2026

Version 1.5.0 ships one change: the migration of the app's
in-process broadcast receivers to a single registrar. The two new
classes — `BroadcastTranslator` and `LlamaMusicPlayer` — replace
fifty-five receivers across ten classes with typed events on a
process-wide EventBus. The change is internal; every user-facing
behaviour the app had in 1.4.8 is unchanged. For the full change
history, see the `[1.5.0]` entry in the [CHANGELOG](CHANGELOG.md).
For technical specifications, installation, and usage, see the
[README](README.md).

---

## What's in this release

### In-process broadcasts now travel through one registrar

Before this release each of the ten receiver-owning classes
registered its own `BroadcastReceiver` instances, unregistered them
in the matching lifecycle method, and dispatched on
`intent.getAction()` inside `onReceive`. That pattern was
duplicated ten times, was invisible to the compiler, and silently
lost events whenever a class forgot to register or unregister in
the right place.

Every in-process receiver is now registered once, by
`BroadcastTranslator`, at process start. The translator converts
each broadcast into a typed event and posts it to the greenrobot
EventBus. The ten classes that used to own receivers now subscribe
to those events with `@Subscribe` methods whose bodies are the
original `onReceive` bodies, unchanged except for the substitution
of `intent.get*Extra("key")` with `event.field`. The
`LlamaMusicPlayer` Application subclass owns the translator's
process lifetime: the translator is started in `onCreate` and, in
the emulator only, stopped in `onTerminate`. (On a production
device `onTerminate` is never called; the framework tears down the
process's receivers with the process itself, so the translator's
lifetime is the process lifetime by construction.)

The migration removed fifty-five receivers, the helper methods
that registered them, and the helper methods that unregistered
them, along with the dead `ACTION_*` constants and the
`BroadcastReceiver` / `IntentFilter` imports those removals
orphaned. The fifty-five receivers map onto thirty stems, because
several stems were received by more than one class (for example,
`THEME_COLOR_CHANGED` was received by four classes). Two receivers
were deliberately retained: `MusicService`'s
`VOLUME_CHANGED_ACTION` receiver, which is a system broadcast that
reads one broadcast and replies on the same turn, and
`MetadataEditor`'s ephemeral `stateReceiver`, which is a
synchronous request/response pattern scoped to a single method.

The compiler now sees every stem. Every event class declares the
fields its sender writes; every subscriber names the event type it
consumes. "Who handles `METADATA_UPDATED`" is one `grep` for
`onMetadataUpdatedEvent` away, and "which fields does
`UPDATE_PLAYER` carry" is one file away.