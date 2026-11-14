# The Origin Story of Llama Music Player

## *A Story of Passion, Partnership, and Persistence*

---

> **A note on this document.** This story is a living record. Llama
> Music Player is still being built, and the story will be updated as
> the project grows.

---

### Prologue: The Developer Who Cannot Write Code

There is a developer in Ghana who cannot write code.

Not from memory. Not from syntax. Not from the muscle memory that
comes from years of training. Hand him a blank editor and ask for a
music player, and his fingers hover over the keyboard, uncertain where
to begin.

And yet, Llama Music Player exists. It plays music. It edits metadata,
manages folders, and runs on Android with no tracking, no analytics,
no ads. It has a full equalizer with sixty presets, a dark theme, and
a metadata editor that lets users update ID3 tags and album art.

How does someone who cannot write code build a music player?

This is how it happened.

---

### Part One: The Seed

I heard stories about the people who built software — the developers
who could make machines do anything. I wondered: *why are they called
developers? What does it mean to develop?*

In school I played Road Rash and Need For Speed SE II on the desktop
and wondered how those games were made. The question stayed with me.
It grew roots.

I loved music, too — not just listening to it, but feeling it. AIMP
was the player I used for years. I would download the latest version
and new skins, watch how the player behaved, and take mental notes.
I used VLC as well. Both served me well.

But something was missing. I wanted to build a music player myself —
not to replace them, but to offer something new. An alternative. A
player that just plays music, with metadata editing features, because
updating metadata for my local files is something I enjoy doing.

One day the thought arrived and would not leave: *let us build a
music player.*

---

### Part Two: The First Step

I did not have a laptop. I did not have Android Studio. I did not have
the knowledge a developer was supposed to have.

What I had was a smartphone running Android 12, a web browser, and an
idea that would not let me sleep.

The best way to learn something is to do it yourself. If you fail, you
learn by finding out why you failed. I read every guide I could find
on how to develop a music player. I did not have what it took to
start, but I had one belief: *I could bring Llama to life.*

The first version was a WebView app — a browser window wrapped in an
APK. Crude and simple, but it played music. It had the basic features
of adding songs and clearing them. And something about the sound
quality attracted me to it: it sounded louder than AIMP through my
Bluetooth speaker. I did not know why. I just knew it did, and that
was enough to keep me going.

I used what I could lay my hands on:

- **APK Builder** for code editing
- **Solid Explorer** for archiving source files
- **Meta AI on WhatsApp** for the initial development
- **Deepseek AI**, which arrived later and became instrumental as the
  codebase grew
- **ChatGPT**, useful when Deepseek got confused with the guidelines

I created a GitHub account after I had completed the project. They say
software development never ends, but it was satisfying, and I wanted
to share my work with the world. GitHub Actions was a necessity:
without a laptop, APK Builder could not build the updated codebase. I
uploaded the entire codebase as a zip archive, set up the workflow,
and created signing keys so the app would be signed automatically.

---

### Part Three: The Partnership

I could not write code, but I could *think*. I could reason. I could
imagine what the app should do, look like, and feel like, and describe
it in words.

I started with Meta AI on WhatsApp. I would send my ideas, my
frustrations, my descriptions. The AI would respond with code. I would
paste the code into APK Builder, read through it, ask questions,
rename variables or classes, tap build, install, and test.

It crashed. A lot.

But each crash was a lesson. I would describe what went wrong, the AI
would offer a fix, and I would try again. Sleepless nights blurred
into weeks. Weeks became months. Deepseek AI arrived, and the
partnership deepened.

#### The Llama Way

I named the app `Llama Music Player` because of how it was developed.

**LLaMA** stands for **Large Language Meta AI** — the name Meta gave
to their AI. But that was not the full story. I also told the AI that
the app could be compared to a real llama, an animal known for being
calm and sociable.

I called my philosophy *The Llama Way*: minimalist, intuitive, no
complications, no settings menu, just a music player that does its
job. I tried to make Llama as intuitive as possible, and I refused to
add tracking or analytics. A clean music player that respects the
user's privacy.

I told the AI the app name was a tribute to our partnership. The AI
could write code; I could reason, imagine, and initiate. Everything I
learned, I learned by practice. If I had not done it this way, Llama
Music Player would not exist as you see it today.

---

### Part Four: The Body of Llama

I asked the AI to arrange the app into the systems of the human body.

The codebase was not generated overnight. The first GitHub release
landed after five months of development, and the project has been
growing ever since. I did not use debugging software, nor did I add
unit tests. I built the app so that I could understand what each file
did, and I chose monolithic classes because that is what I could work
with. I made the design and functionality decisions, and I asked the
AI to implement each feature. I asked it to write the code
professionally, following Google's recommended guidelines and best
practices for Android.

The architecture mirrors a living organism. Every class is a part of
the body, and every part has a job it does and no other.

- **The heart** — `MusicService` pumps the audio. `PlaybackWindow` is
  the blood, carrying the current transition to every surface.
  `AlbumArtCarousel` is the pulse, showing the transition on the album
  art. `PlaybackScope` and `PlaybackScopeDialog` describe and choose
  the subset of the library the heart plays from.
- **The lungs** — `PlaylistManager` breathes life into the playlist.
- **The nervous system** — `NavigationController` coordinates actions
  across the app. `MusicPlayer` is the brain, the main surface the
  user sees.
- **Memory** — `MusicLibrary` is the working memory. `SongDatabase` is
  the long-term memory. `AlbumArtCache` is the visual memory. `Song`
  is a single memory, one song and its facts.
- **The senses** — `EqualizerManager` shapes the sound.
  `StorageObserver` listens for changes to the music folder.
  `CharacterMapper` decides what a tag means when it is missing.
- **The reflex arc** — `BluetoothReceiver` reacts to a headset
  connecting or disconnecting.
- **The hands** — `MetadataEditor` lets the user modify ID3 tags.
  `MetadataReader` pulls tags out of a file. `MetadataWriter` puts
  tags back without corrupting the file. `StorageAccessHelper` holds
  the permissions that let the hands reach the files.
- **The voice** — `ToastManager` speaks to the user inside the app.
  `NotificationController` speaks from the notification and the lock
  screen.
- **The skin** — `ThemeManager` is the current theme. `ThemeHelper`
  paints every control.
- **The organs the user touches** — `PlaylistActivity`,
  `EqualizerActivity`, `FolderManager`.
- **The face** — `CreditsDialog`, `LanguageDialog`, `PresetDialog`,
  `ThemeDialog`.
- **The fingers** — `PlaylistAdapter` and `FolderAdapter` render each
  row.
- **The immune system** — `PermissionChecker` decides what the app is
  allowed to touch.
- **The skeleton** — `FontAwesome` is the alphabet of shapes.
  `LanguageManager` is the tongue.
- **The eyelid** — `OverlayManager` dims the view while work is
  happening.
- **The edge** — `EdgeScrollHelper` is the second sense in the
  fingers, the proportional fast-scroll on a long list.

Every class has a purpose. I asked the AI to document the codebase so
I could better understand it and have context for my requests during
debugging. The metric was always: does this feature work, does it
crash the app, does anything look or feel weird.

---

### Part Five: The Testing Process

To test the app, I did what I knew best:

1. Build it.
2. Install it.
3. Run it.
4. If I noticed anything odd, or saw something that did not sit
   right, I would describe the issue to the AI, provide the code, and
   ask it to first understand the current code and identify the
   problem before suggesting the best solution.
5. Repeat.

The AI would often truncate its code and assume I would fill in the
rest. This was frustrating. I would say: *this project is not a joke,
it is a real idea I am trying to bring to life. Please do not truncate
the code. Give me the complete updated class.* And slowly, it learned
to respond fully.

I learned not by rote or muscle memory, but through the experience,
the failures, and the crashes.

---

### Part Six: The Dream

I have a dream: to have a laptop, a powerful one, so I can learn
software development properly.

Llama exists now. Llama plays music. It is not the best or perfect,
but it is my idea made manifest. It is proof that anyone can build
software, even with limited resources.

If my story has moved you, and if you believe in the power of passion,
partnership, and persistence, please consider sponsoring my journey. A
laptop would mean a lot to me.

**Support my journey:** [https://github.com/sponsors/oneadzido](https://github.com/sponsors/oneadzido)

---

### Coda

I see a future where I have a laptop, where I can learn software
development properly, where I can build Llama Music Player even
better, and where I can help others who, like me, have dreams but not
resources.

It all begins with a thought...

***Richard Korbla Adzido, 2026***

---

### About the Developer

Richard Korbla Adzido is a self-taught, AI-assisted software developer
from Ghana.

Driven by a lifelong passion for computers and music, he began
building on an Android smartphone, without a laptop or formal
programming training. Working alongside AI, he turned a simple idea
into a feature-rich Android music player through experimentation,
hands-on testing, problem-solving, and continuous improvement.

His philosophy, The Llama Way, is simple: keep the experience
intuitive, avoid unnecessary complications, and respect the listener's
privacy.

He built with the tools he had, learned along the way, and turned his
vision into reality.

**Support his journey:** [https://github.com/sponsors/oneadzido](https://github.com/sponsors/oneadzido)

---

*End of Origin Story*