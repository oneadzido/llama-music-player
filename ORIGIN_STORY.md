# The Origin Story of Llama Music Player

## *A Story of Passion, Partnership, and Persistence*

---

### Prologue: The Developer Who Cannot Write Code

There is a developer in Ghana who cannot write code.

Not from memory. Not from syntax. Not from the muscle memory that comes from years of training in classrooms or bootcamps. If you hand him a blank editor and ask him to write a music player from scratch, his fingers hover over the keyboard, uncertain where to begin.

And yet, Llama Music Player exists. It is real. It plays music. It edits metadata, manages folders, and runs on Android devices with no tracking, no analytics, no ads. It has a full equalizer with sixty presets, a beautiful dark theme, and a metadata editor that lets users update ID3 tags and album art.

How does someone who cannot write code build a music player?

This is the story of how it happened...

---

### Part One: The Seed

#### *"Who is a developer?"*

This story began with a child in Ghana who loved computers. That child was me. I am Richard Korbla Adzido, and this is how Llama Music Player came to life.

I heard stories about the people who built software, the developers who could make machines do anything. I wondered: *Why are they called developers? What does it mean to develop?*

Back in school, I played Road Rash and Need For Speed SE II on the school desktop. I always wondered *how those games were made.*

Those questions stayed with me. They grew roots.

I imagined the cost of building software, the complexity of the code, the skill required. I wondered: *Could I ever do that?*

#### *I Love Music...*

Not just listening to it, but feeling it. I loved AIMP, the music player I had used for years before I built Llama Music Player. I would always go to the developer's website and download the latest version and some skins too. I observed the player's behavior, and took mental notes of how the app behaved.

I also used VLC media player — both are great players that have served me well. I take nothing from them.

But something was missing. I wanted to build a music player myself, one that could stand alongside these giants. Not to replace them, but to offer something new. An alternative.

I wanted to build something that just plays music, with metadata editing features — something I enjoy doing, updating metadata for my local music files. I came to understand what a music app should do: It should simply play music.

One day, a thought came to me: *Let's build a music player...*

---

### Part Two: The First Step

#### *A Phone, A Browser, A Dream*

I did not have a laptop. I did not have Android Studio. I did not have the knowledge a developer was supposed to have.

What I had was a smartphone running Android 12, a web browser, and an idea that would not let me sleep.

*The best way to learn something is to do it yourself. If you fail, you learn by finding out why you failed.*

I tried to find another way, using my smartphone to build the app. I checked guidelines on how to develop a music player. I read them but I did not have what it took to actually start. But I had one belief: *I could bring Llama to life, not just a thought.*

#### *The First Version*

The first version of Llama Music Player was a WebView app. A browser window wrapped in an APK. Crude. Simple. But it played music.

It had the basic features of adding songs and clearing them.

And something about the sound quality attracted me to the app. It sounded louder than the output from AIMP when connected to my Bluetooth speaker.

I did not know *why* it sounded louder. I just knew it did. And that was enough to keep me going.

#### *My Tools*

I used what I could lay my hands on:

- **APK Builder** for code editing
- **Solid Explorer** for archiving source files
- **Meta AI on WhatsApp** for the initial development
- Then **Deepseek AI** (released later, I got to know and started experimenting my idea of Llama Music Player with it)
- Also **ChatGPT** (useful when Deepseek AI got confused with the guidelines)

I created a GitHub account as a result — this was after I had completed the project (well, they say software development never ends, but it was satisfying and I wanted to share my work with the world).

My GitHub Actions was a necessity to build the app properly for Android since I did not have a laptop, and APK builder couldn't build the updated codebase.

I therefore upload the entire codebase as a zip archive and use GitHub Actions to build the app. I created signing keys to automatically sign the app.

---

### Part Three: The Partnership

#### *"I can't write code, but AI can."*

I could not write code, but I could *think*. I could reason. I could imagine what the app should do, look and feel like, and describe it in words.

I started initial development with Meta AI on WhatsApp. I would send my ideas, my frustrations, my descriptions of how the app should look and feel like. The AI would respond with code.

I would paste the code into APK Builder (my code editor), read through it, ask questions, rename some variables or classes, tap build, install the app, and test it.

It crashed. A lot.

But each crash was a lesson. I would describe what went wrong, the AI would offer a fix, and I would try again. Sleepless nights blurred into weeks. Weeks became months.

Then Deepseek AI arrived, and the partnership deepened. As the codebase grew, and more classes were added, Deepseek AI became instrumental in my journey.

#### *The Llama Way*

I named the app `Llama Music Player` because of how it was developed.

**LLaMA** stands for **Large Language Meta AI** — the name Meta gave to their AI.

But that was not the full story. I also told the AI that the app could actually be compared to a real llama, an animal that is known to be calm and sociable.

I called my philosophy *The Llama Way*.

Minimalist. Intuitive. No complications. No settings menu. Just a music player that does its job. That is the Llama Way.

I heard that the best software does not make the user think. I tried to make Llama as intuitive as possible. And no tracking, analytics, etc. — just a clean music player that respects the user's privacy.

#### *"A tribute to my partnership with AI."*

I found a way to partner with AI, telling it that this project was a real idea I was trying to bring to life. I explained the app name to the AI, saying that I named it so — as a tribute to my partnership with AI.

I could not write code, correct syntax, etc., but AI could. I could however think, reason, and come up with ideas, and initiate prompts.

I realized the AI was a tool I could use to achieve my goal. Everything I learned was by practice, at best.

My way of developing the software was the way I knew. I knew there were other ways but this was what I could do.

If I had not done it that way, `Llama Music Player` would not exist as you see it today.

---

### Part Four: The Body of Llama

#### *A System Like the Human Body*

I asked the AI to segregate the app into the systems in the human body.

The current codebase was not something that was generated overnight. It took me 5 months, and still counting, sleepless nights.

I did not use debugging software, nor added unit tests. I chose to build the app in such a way that I could understand what each file did.

So I decided to use monolithic classes. I made the decisions for the app's design and functionality, and asked the AI to implement each feature.

I asked the AI to write the code professionally, following Google's recommended guidelines and best coding practices for Android music app development.

#### *The Architecture*

I designed Llama like a living organism:

- **MusicService** is the heart — pumping audio through the app
- **MusicLibrary** is the memory — storing songs and metadata  
- **EqualizerManager** is the senses — shaping the sound
- **NavigationController** is the nervous system — coordinating actions
- **ToastManager** is the voice — communicating with the user
- **ThemeManager** is the skin — the color of Llama
- **SongDatabase** is the backbone — persistent storage of metadata
- **MetadataEditor** is the hands — allowing users to modify ID3 tags
- **PlaylistService** is the lungs — breathing life into the playlist
- **StorageObserver** is the ears — listening for changes to the music library

Every class has a purpose.

I asked the AI to document the codebase for clarity. It helps me to understand the codebase, so I can better have a context for my requests during debugging (visually, through actual use of the app).

The metric is: *Does this feature work? Does it crash the app? Does anything look or feel weird?*

---

### Part Five: The Partnership Deepens

#### *"Give me your best."*

To ensure the app got better over time, I asked the AI to give its best.

The AI would often give me code snippets and assume I would fill in the rest, truncating the code. This was frustrating. But I found a way to work with it.

I would say: "This project is not a joke. It's a real idea I'm trying to bring to life. Please don't truncate the code. Give me the complete updated class."

And slowly, the AI learned to respond fully.

#### *The Testing Process*

To test the app, I did what I knew best:

1. First build it
2. Install it
3. Run it
4. If I noticed anything odd or saw something that did not sit or feel right with me, I would try and describe the issue to the AI
5. Provide the code
6. Ask the AI to do its best to understand first, the current code, identify the problem, before suggesting the best solution

I learned not by rote or stored in muscle memory but through the experience, the failures, and the crashes.

---

### Part Six: The Dream

#### *"To Have a Laptop."*

I have a dream: to have a laptop, a powerful one, so I can learn software development properly.

Llama exists now. Llama plays music. It is not the best or perfect but it is my idea made manifest.

I have given you Llama—a music player built from a phone, a browser, and a partnership with AI. It is not perfect, but it is real. It is proof that anyone can build software, even with limited resources.

If my story has moved you, and if you believe in the power of passion, partnership and persistence, please consider sponsoring my journey. A laptop would mean a lot to me in my software development journey.

**Support my journey:** [https://github.com/sponsors/oneadzido](https://github.com/sponsors/oneadzido)

---

### Epilogue: Lessons from the Journey

I learned all these things on my journey to building Llama Music Player:

- How to use GitHub Actions for CI/CD
- How to create signing keys
- How to build for Android without a laptop
- How to debug without debugging tools
- How to persist through failure
- How to partner with AI
- How to design for intuition

Everything I learned was by practice.

---

### Coda: The Vision

I see a future where:

- I have a laptop
- I can learn software development properly
- I can build Llama Music Player even better
- I can help others who, like me, have dreams but not resources

**It all begins with a thought...**

***Richard Korbla Adzido, 2026***

---

### Postscript: Anyone Can Build Software

If you read the codebase of Llama Music Player, you will see:

- A music player that plays music
- Metadata editing features
- A full equalizer with 60 presets
- Folder management
- No tracking, no analytics
- Clean, intuitive design
- A tribute to the partnership between a human mind and AI

Llama Music Player is proof that anyone can build software, even with limited resources.

---

### About the Developer

Richard Korbla Adzido is a self-motivated, self-learning developer from Ghana. He has always been passionate about computers since he was a child. He built Llama Music Player using his smartphone, a browser, and a partnership with AI.

He is not a developer by training. He is a developer by passion.

He offers you Llama Music Player.

**Support his journey:** [https://github.com/sponsors/oneadzido](https://github.com/sponsors/oneadzido)

---

*End of Origin Story*