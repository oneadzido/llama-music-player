# The Origin Story of Llama Music Player

## *A Story of Passion, Partnership, and Persistence*

---

### Prologue: The Developer Who Cannot Write Code

There is a developer in Ghana who cannot write code.

Not from memory. Not from syntax. Not from the muscle memory that
comes from years of training. If you hand him a blank editor and ask
him to write a music player from scratch, his fingers hover over the
keyboard, uncertain where to begin.

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

Back in school, I played Road Rash and Need For Speed SE II on the
school desktop. I always wondered how those games were made. Those
questions stayed with me. They grew roots.

I loved music, too. Not just listening to it, but feeling it. AIMP
was the player I used for years. I would go to the developer's website
and download the latest version and some skins. I watched how the
player behaved and took mental notes. I used VLC as well. Both are
great players that have served me well.

But something was missing. I wanted to build a music player myself —
not to replace them, but to offer something new. An alternative. A
music player that just plays music, with metadata editing features,
because updating metadata for my local files is something I enjoy
doing.

One day, a thought came to me: *let us build a music player.*

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

The first version was a WebView app. A browser window wrapped in an
APK. Crude and simple, but it played music. It had the basic features
of adding songs and clearing them. And something about the sound
quality attracted me to it — it sounded louder than the output from
AIMP when connected to my Bluetooth speaker. I did not know why. I
just knew it did, and that was enough to keep me going.

I used what I could lay my hands on:

- **APK Builder** for code editing
- **Solid Explorer** for archiving source files
- **Meta AI on WhatsApp** for the initial development
- **Deepseek AI**, which arrived later and became instrumental as the
  codebase grew
- **ChatGPT**, useful when Deepseek got confused with the guidelines

I created a GitHub account after I had completed the project — they
say software development never ends, but it was satisfying and I
wanted to share my work with the world. GitHub Actions was a
necessity: without a laptop, APK Builder could not build the updated
codebase. I uploaded the entire codebase as a zip archive, set up the
workflow, and created signing keys so the app would be signed
automatically.

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

The codebase was not generated overnight. Initial release took five
months, and many sleepless nights. I did not use debugging
software, nor did I add unit tests. I built the app so that I could
understand what each file did, and I chose monolithic classes because
that is what I could work with. I made the design and functionality
decisions, and I asked the AI to implement each feature. I asked it to
write the code professionally, following Google's recommended
guidelines and best practices for Android.

The architecture mirrors a living organism. Every class is a part of
the body, and every part has a job it does and no other.

#### The heart and its circulation

- **MusicService** — the heart. Pumping audio through the app.
- **PlaybackWindow** — the blood. Carrying the current transition to every surface.
- **AlbumArtCarousel** — the pulse. Showing the transition on the album art.

#### The lungs

- **PlaylistManager** — the lungs. Breathing life into the playlist.

#### The nervous system

- **NavigationController** — the nerves. Coordinating actions across the app.
- **MusicPlayer** — the brain. The main surface, coordinating everything the user sees.

#### Memory

- **MusicLibrary** — the working memory. The songs loaded for this session.
- **SongDatabase** — the long-term memory. The persistent store.
- **AlbumArtCache** — the visual memory. The decoded album art.
- **Song** — a single memory. One song and its facts.

#### The senses

- **EqualizerManager** — the ears. Shaping the sound.
- **StorageObserver** — the listening. Waiting for changes to the music folder.
- **CharacterMapper** — the reading. Deciding what a tag means when it is missing.

#### The reflex arc

- **BluetoothReceiver** — the reflex. Reacting to a headset connecting or disconnecting.

#### The hands

- **MetadataEditor** — the hands. Allowing the user to modify ID3 tags.
- **MetadataReader** — the reading hand. Pulling tags out of a file.
- **MetadataWriter** — the writing hand. Putting tags back without corrupting the file.
- **StorageAccessHelper** — the grip. Holding the permissions that let the hands reach the files.

#### The voice

- **ToastManager** — the voice. Speaking to the user inside the app.
- **NotificationController** — the outward voice. Speaking from the notification and the lock screen.

#### The skin

- **ThemeManager** — the skin's color. The current theme.
- **ThemeHelper** — the skin's touch. Painting every control.

#### The organs the user touches

- **PlaylistActivity** — the playlist organ.
- **EqualizerActivity** — the equalizer organ.
- **FolderManager** — the folder organ.

#### The face

- **CreditsDialog** — the face. Telling the user what the app is.
- **LanguageDialog** — the speaking face. Choosing the language.
- **PresetDialog** — the equalizer's face. Choosing a preset.
- **ThemeDialog** — the skin's face. Choosing a theme.

#### The fingers

- **PlaylistAdapter** — the finger on the playlist. Rendering each row.
- **FolderAdapter** — the finger on the folder list. Rendering each row.

#### The immune system

- **PermissionChecker** — the immune system. Deciding what the app is allowed to touch.

#### The skeleton

- **FontAwesome** — the alphabet of shapes. The icons.
- **LanguageManager** — the tongue. The languages the app can speak.

#### The eyelid

- **OverlayManager** — the eyelid. Dimming the view while work is happening.

Every class has a purpose. I asked the AI to document the codebase so
I could better understand it and have context for my requests during
debugging. The metric was always: does this feature work, does it
crash the app, does anything look or feel weird.

---

### Part Five: The Testing Process

To test the app, I did what I knew best:

1. Build it
2. Install it
3. Run it
4. If I noticed anything odd, or saw something that did not sit
   right, I would describe the issue to the AI
5. Provide the code
6. Ask the AI to first understand the current code and identify the
   problem before suggesting the best solution

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

It all begins with a thought.

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