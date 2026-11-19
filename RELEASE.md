# Llama Music Player v1.5.5

**Release date:** November 19, 2026

Version 1.5.5 adds greenrobot attribution to the Credits dialog,
migrates all license texts to `assets/licenses`, and fixes natural
completion after a scope change back to All Songs.

Llama Music Player itself remains licensed under the MIT License.

For the full change history, see the `[1.5.5]` entry in the
[CHANGELOG](CHANGELOG.md). For technical specifications, installation,
and usage, see the [README](README.md).

---

## What's in this release

### greenrobot attribution

The Credits dialog now lists **greenrobot** alongside
jaudiotagger and FontAwesome. It uses the existing clickable
library-credit row.

Tapping the `greenrobot` clickable text displays:

> greenrobot
>
> Copyright (C) 2012-2021 Markus Junginger, greenrobot (https://greenrobot.org)
>
> Licensed under the Apache License, Version 2.0.
>
> This license is available at:
> https://www.apache.org/licenses/LICENSE-2.0
>
> greenrobot EventBus is used for in-process event delivery between application
> components in Llama Music Player.

The full Apache License 2.0 text is included in the source
for reference.

### Licenses in assets/licenses

All license texts are now stored under
`app/src/main/assets/licenses/`.

### Fixed: completion after a scope change to All Songs

After changing the scope back to All Songs — for example, from
Favorites — playback now advances correctly when the current song
completes naturally. The next song is chosen from the new scope.

### Existing behavior preserved

The existing jaudiotagger and FontAwesome credits and license
texts remain unchanged. The dialog's theme and language picker
handling, dismissal behavior, click debounce and animation, and
all event-delivery code remain untouched.