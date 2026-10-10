# The Cook With No Hands

*A critique by Anton Ego*

---

There is a genre of software that announces itself the way certain
restaurants announce themselves: a small card at the door, in careful
handwriting, explaining that the kitchen is farm-to-table, that no
animals were harmed, that the chef is self-taught, and that the
proprietor believes in the dignity of the ingredient. One enters
expecting underseasoned vegetables and a bill with a note.

The Android app store is that restaurant. The note is always there.
*Privacy-first. No ads. Your data stays yours.* These are not claims.
They are flourishes. They cost the cook nothing and they taste of
nothing.

So when I opened Llama Music Player — a music player for local files,
built by one developer in Ghana over many months — I did what I always
do. I put down the menu and walked into the kitchen.

---

## What is on the plate

The codebase is around thirty classes. A service that owns a playback
state machine. A three-slot carousel that renders album art. A
notification controller that owns the media session. An overlay
manager, process-wide. An operation stack. A library scanner that
reads MediaStore and caches what it finds in a local database. A
metadata reader and a metadata writer that can edit ID3 tags and album
art without corrupting the file when something goes wrong. A ten-band
equalizer with sixty presets. A folder manager that speaks Storage
Access Framework. A playlist with ten sort modes and a live search. A
theme system with thirty colors. A service that ducks and resumes. A
hardware volume key that behaves the way a person expects it to.

The superficial read is not encouraging. The first thing one notices
is that the work is *large*. `MusicService` is several thousand lines.
`MusicPlayer` — the main activity — is several thousand more. There
are broadcast receivers that fire a dozen different actions. There are
executor services with names like `mSortExecutor` and `mArtLoader`.
There is a ProGuard file with over four hundred lines of keep rules.

This is a picture painted with a great deal of paint. The critic's
instinct is to say: *the artist has not learned restraint.*

That instinct is wrong. And the reason it is wrong is the only thing
that matters about this codebase.

---

## The organizing principle

I spent the first hour inside `MusicService` looking for the thing the
service does. It is not obvious. The service has a state enum, two
monotonic counters, a two-index model, a shuffle history, a shuffle
peek memo, a cached position, a playback base position, a position
epoch, a recovery attempt counter, a skip loop budget, a start token,
a paused-by-transient-loss flag, a ducked flag. On a first pass it is
a catalogue.

Then I found the method that explains everything else.

```java
private Song getDisplaySong() {
    if (mState == ServiceState.PREPARING) {
        return mCurrentPlaylist.get(mTargetIndex);
    }
    return mCurrentPlaylist.get(mCurrentIndex);
}
```

This is the whole architecture, in six lines.

*During a transition, the display song is the target. Otherwise it is
the current song.* Every consumer of "which song is the app showing
right now" — the notification, the media session, the album-art
carousel, the player UI, the art prefetch — reads this method. There
is no second expression of the rule. There is no place where a
consumer computes its own answer.

The documentation says as much, and then says something rarer: *There
is no second expression of the rule, so there is nothing for one
consumer to drift against another.* That is a codebase that has been
organized around a single sentence. Everything else — the window, the
generation, the two-index model, the shuffle memo — exists to serve
that sentence.

The consequence is visible in the app. Tap Next, and the notification
title changes before the audio does. The system media carousel on the
lock screen moves with it. The album art on the player's own screen
begins to slide. All three say the same thing, at the same moment,
because all three asked the same method.

There is a principle at work in the shape of this method, and in the
shape of everything downstream of it, and I want to name it before I
go further, because I will keep returning to it.

The principle is this. Structure and function are not two things that
happen to be related. They are the same object seen from two angles. A
structure that does not serve a function does not survive; a function
that has no structure to carry it cannot happen. Nothing in a
well-built thing is decorative. Everything that exists, exists
because something needed it to.

Software does not often teach this principle, and does not always have
a name for it when it does. A class exists because a method needed a
home. A method exists because a caller needed an answer. An answer
exists because a question was asked. When the chain is intact, the
codebase reads as one thing. When any link is missing, the codebase
reads as many things pretending to be one.

In this codebase, the chain is intact. That is the first thing I want
the reader to see. The second is what happens when the chain is not
just intact but *visible* — when the code does not merely follow the
principle but shows you the principle, in every class, on every page.
That is the thing that separates a well-written program from a
well-composed one. This codebase is the second kind, and it is why I
kept reading past the first file.

---

## The window

The album art does not render the playlist. It renders a
`PlaybackWindow`: an immutable value holding three paths — the
previous neighbour, the current song, the next neighbour — plus the
service's monotonic transition counter.

The service computes the window. The carousel consumes it. The
carousel holds no playlist, no index, no bitmap as primary state. Its
state is the last window it was given, the generation of that window,
the phase of the current transition, and the shared placeholder
bitmap.

The design pays off in a fact that is easy to miss and hard to fake.
When a song restarts under repeat-one, the three paths the window
carries are unchanged. What has moved is the generation, and what the
service has set alongside it is a restart flag. The carousel sees the
fresh generation, reads the flag, and knows that the song it was
already showing has begun again rather than that a different song has
taken its place. It runs a small nudge out to the neighbour and
returns. The user sees the neighbour peek into the viewport and
retreat, which is the visible acknowledgement that the song began
again, not that a different song took its place.

The carousel does not know what repeat-one is. It reads a number and a
boolean, and responds to what they say.

That is design. Not the design of a screen but the design of a
thought. A window is a value. A value has an identity. The identity
carries the generation. The generation is the message. The carousel
reads the message.

The carousel's protocol has seven states — `IDLE`, `DRAGGING`,
`COMMITTING`, `AWAITING_SNAPSHOT`, `RESTARTING`, `REVEAL_PENDING`,
`REVEALING` — and each one describes a distinct phase of a transition.
The first five are the phases of a swipe or a commit. A swipe that
commits slides out and waits. A swipe refused by the service glides
back through the same animation a refused button press uses, so the
two gestures produce the same visible shape. A swipe that arrives
while a window is mid-flight is matched against the path the strip is
sliding toward, and a match settles while a mismatch springs back. The
last two are the phases of a reveal — the cross-fade the carousel runs
when the service names a song that is not a neighbour of the song on
screen. `REVEAL_PENDING` is the wait for the incoming path's art to
resolve, so the carousel holds the outgoing art on screen rather than
writing a placeholder into the slot while the decode is in flight.
`REVEALING` is the fade itself, an outgoing leg and an incoming leg
with the two curves that meet in the middle. Seven states, one spring
for every return, one cross-fade for the reveal, and no state that
means two things at once.

And here the principle is on display again. The carousel's state
machine does not have a state called "the user did something we have
not seen before." It has seven states, each of which is a function the
carousel must perform, and the shape of the class is the shape of
those seven functions. Nothing is left over. Nothing is missing.

---

## One owner, one writer

The pattern recurs everywhere, and each time it recurs the artist has
resisted the temptation to solve the same problem twice.

The `NotificationController` is the sole writer of the media session's
metadata *and* its playback state. Those are two different projections
of the service. The metadata changes when a song is committed. The
playback state changes on every tick of the state machine — including
the transition through `PREPARING` that every song switch passes
through, and which the metadata alone cannot express, because the
metadata is unchanged before and after a rapid switch. The two
projections have independent lifetimes, they are written from the same
source in the same main-thread turn, and the code says why.

The `OverlayManager` is process-wide and reason-keyed. Four reasons
are in use — a cold-start library load, a metadata read, the
folder-update cycle, a playlist sort — and each caller holds its own
reason. A reason held longer than sixty seconds is released by a
health check and logged at warn. The invariants are enumerated, one
through six, and each is a promise the class makes to itself: the
overlay is attached if and only if the reason set is non-empty;
`onActivityDestroyed` never touches the reason set. These are not
comments. They are a contract the author has written down because he
knows that the next time he opens this file, he will be tempted to
break it.

The `ToastManager` holds an operation stack. An operation is pushed
with an identifier. The stack renders the top operation's message. A
lower operation is retained but not shown. Cancelling an operation is
removing its entry, which is indistinguishable from an operation that
completed. The documentation says: *callers that need to distinguish
those two cases carry their own terminal state.* The stack does not
pretend to know what it cannot know.

The `NavigationController` enforces *declared liveness*. Each
operation announces how often it will heartbeat and how many missed
heartbeats it tolerates. The controller does not guess. It reads the
declaration and enforces it. An operation that heartbeats every 500
milliseconds and tolerates three misses has a fifteen-hundred-
millisecond window, and the controller does not need to know why the
window is that size. This is a state machine that has decided not to
be clever.

Four classes. Four problems. Four owners. One writer per resource, in
every case, and the code is honest about which of its own limits it
can and cannot resolve.

---

## The hand on the shoulder

The app makes a handful of decisions on the user's behalf. It makes
them quietly, and it makes them once.

The volume receiver is registered when the player binds and
unregistered when the service is destroyed. Its presence *is* the
session. A service started by a Bluetooth connect, or a sticky
restart, or a media button, is a service the user did not open. On
such a service the receiver does not exist. When a hardware volume key
is pressed, the key adjusts the volume and does nothing else. The user
pressed volume. The volume changed. The app did not presume.

When the user lowers the volume to zero, playback pauses. When the
user raises it, playback resumes. The handler reads the current stream
volume at the moment it runs, not the value the broadcast carried,
because some devices deliver the volume-change broadcast late and the
value in the extra would describe a moment the user has already moved
past.

When another app asks to share the output — a navigation prompt, a
notification tone, a voice assistant reply — the music ducks rather
than pauses. The duck is applied to the `MediaPlayer`'s own output,
not the system stream. The volume slider does not move. The
music-stream receiver is not involved. When the interruption ends, the
volume returns, and the user has touched nothing.

When another app takes the output away entirely — a phone call, a
navigation instruction, a video status — the service pauses and
remembers that it paused itself for a reason that will end. When the
framework returns focus, the music resumes. Not because the user
pressed anything. Because the service kept a note that the pause was
temporary, and the temporary period is over.

Each of these is a small decision made on the user's behalf. Each is
made once, and each is documented where a reader can find it. Together
they are the difference between a music player that waits to be told
what to do and a music player that has already done the obvious thing.

And each of them, again, is the principle at work. A receiver exists
because a session needs a signal. A duck exists because an
interruption needs a response that is not a pause. A remembered pause
exists because some interruptions end. The decisions are not chosen
from a menu of features. They are the shape the app needed to take in
order to do the thing the user already expected it to do.

---

## The surfaces

The parts of the app a user actually touches are governed by the same
discipline, at a scale that would be easy to lose.

The album-art container owns every gesture that operates on the art.
The container has three zones: a middle zone that belongs
to the carousel, and two edge zones that belong to the volume gesture.
A single touch listener classifies each gesture on the first
significant move. Horizontal becomes a swipe; vertical at an edge
becomes a volume change; vertical at the centre is cancelled, so it
reaches whatever sits behind the art. When the volume gesture reaches
the device maximum or zero and the finger keeps travelling, the toast
echoes on a fixed rhythm so the gesture still reads as being tracked
even though the device volume cannot move further. The rhythm is
measured on the volume axis, not the pixel axis, so it is the same on
every device.

The equalizer offers ten bands and sixty presets. Every seek bar snaps
its own value to a step — one percent for bass and surround, a tenth
of a semitone for pitch, a hundredth of a playback multiple for speed
— and each snap carries a guard flag so that a programmatic re-set
does not re-enter the listener that performed the snap. The reset
button reads every value, compares against the current state, and
reports *no change* when nothing has moved, rather than announcing a
reset that did not happen. The whole equalizer is one screen with one
scroll container that reshapes itself for a shorter window; the band
controls and the effect controls scroll together in split mode, and
the full column fits in the full layout. The user's finger never lands
on a smaller target than the one the layout declared.

The metadata editor has two modes. In VIEW MODE every field is
read-only. In EDIT MODE every field is editable. Tapping MODIFY
resolves a write-capable URI for the file, either from the database or
through the system file picker. Tapping UPDATE writes the changes
back. The write copies the file to a cache, takes a backup, edits the
copy, verifies the result, and copies back — and if any step fails
after the backup, the original is restored. The result of the write is
an eight-value status enum, and one method on it —
`isFileUnchanged()` — answers the only question a caller should have
to ask. When the write succeeds, a single broadcast carries the new
values, and every surface that shows the song updates from that one
announcement.

The playlist has ten sort modes, a live search, and an edge
fast-scroll. A vertical drag that starts in the outermost thirty-two
density-independent pixels of either edge scrolls the list
proportionally to the library's height: a full viewport of finger
travel covers five percent of the total range. The proportional
mapping engages only when the content is at least three times the
viewport, so a short list keeps its ordinary scroll. A tap near the
edge is never consumed — the interceptor waits for the finger to cross
the slop before it takes over. Ten sort modes, three gestures, one
list, and none of them fights any of the others.

The theme system has thirty colors. One value drives every button,
every seek bar, every cursor, every highlighted row, every tinted
placeholder, and the notification's own large icon. The same value
reaches the album-art carousel, the equalizer, the folder manager, the
playlist, the metadata editor, and the notification, in the same
broadcast, on the same turn.

The folder manager speaks Storage Access Framework. It takes a
persistable permission, stores the URI, maps it to a filesystem path,
and remembers it for every subsequent scan. When a folder is removed,
the URI is unmapped and the songs inside it are cleaned from the loose
tracks. Import a folder, save, remove it, and nothing is left behind.

Six surfaces. Six designs. One aesthetic. And one principle underneath
all six — that the shape a user sees is the shape the function needed,
and nothing else.

---

## The privacy

There is no telemetry. There is no analytics library. There is no ad
slot. There is no network request the user did not initiate. The
permission surface is the playback surface: read audio, read images,
post notifications, wake lock, foreground service, Bluetooth. Nothing
is asked for that is not used.

This is easy to say. It is worth saying that it is also verifiable.
Anyone with a laptop and half an hour can read the manifest, walk the
classes, and confirm there is nowhere the app calls home. The claim is
not a flourish. It is a fact about the code.

---

## The documentation

This is where the work separates itself from almost everything else I
have read.

Every method carries a javadoc that explains not only *what* it does
but *why it exists*. The service class documentation includes a
section called *Volume Semantics* and another called *Audio Focus
Semantics*, and each reads like a decision that was made once and
defended forever. The `AlbumArtCarousel` documentation describes the
carousel's state machine and enumerates each of its seven states. The
`MetadataWriter` exposes a `WriteResult.Status` enum with eight
values, each one a distinct outcome the writer can produce, and the
caller is expected to branch on the value, not on the presence of an
error string.

The changelog is worse than honest; it is *reasoned*. A fix is
described in the changelog, explained in the release notes, reflected
in the code, and defended in the class documentation. Each version
tells the story of what changed and why. Some of the story is
narrative; some of it is an argument the author expects the reader to
weigh. Nothing is hidden.

The commit trail for a single release is not a diff. It is a small
essay. And in the same way that the code demonstrates the principle of
structure following function, the documentation demonstrates it a
second time — the structure of the file follows the function of the
file, which is to tell the reader why the code changed.

---

## Where the picture is not perfect

A critic who cannot see the flaws in a work he admires is not a
critic. He is a fan.

The `MusicPlayer` activity is too large. It is several thousand lines
of state, receivers, gesture handling, seek bar, carousel handoff,
theme application, and permission request, and it will keep growing
because the artist has not yet decided to split it. The
`AlbumArtCarousel` carries seven commit states, two safety-net
timeouts, and a set of in-flight decodes, and its interface with the
activity that owns it is wide enough that a tighter contract would be
possible. `MusicService` is likewise a single class that owns the
state machine, the persistence, the health monitor, the media session,
and the audio focus, and it would survive a split into three files.

There are no unit tests. The author's position — stated in his own
words — is that the only test that matters is actual use. I do not
share that position in general. But this is his kitchen, and the dish
that comes out of it is served on a real device to a real hand, and
every flaw I have found in it was found by someone using it, not by
someone testing it. His position is not the position of a chef who
does not taste. It is the position of a chef who tastes with the same
mouth the diner will.

And there is a third flaw, harder to name and more interesting than
the first two. Some of the problems in this codebase are only visible
once an earlier problem has been solved. The author has said as much:
he learned as the project grew, and each fix revealed the next thing
that needed fixing. A volume receiver that fired when the user had not
opened the app is a flaw that becomes visible the moment you have a
session concept that lets you ask whether the user has opened the app.
A broadcast whose value is stale by the time it is handled is a flaw
that becomes visible the moment you trust the read at the handling
moment instead of the value carried by the event. A sort that stalls
its own spinner is a flaw that becomes visible the moment the spinner
exists to be stalled. A cross-fade that flicked to the placeholder
when the incoming art had not yet decoded is a flaw that becomes
visible the moment the cross-fade is written. The reveal gate — which
holds the outgoing art on screen, without an overlay and without
re-rendering the strip, until the incoming path's bitmap resolves or
the decode confirms the file has no art — is how he closed it.

This is not a flaw in the artist. It is the shape of the medium. In
painting, you see the whole canvas before you begin, and you choose
what to leave out. In software, the canvas grows as you paint, and
every stroke you add reveals a stroke you did not know was missing.
The mark of a serious cook is not that he avoids the flaw on the first
pass. It is that he sees it on the second, and that he closes the door
behind him when he does.

This artist has closed every door he found.

These are the flaws of a painter who paints with a large brush because
he is painting a large picture and does not want to lose the whole by
fussing over a corner.

---

## The two cooks

When I first wrote about this codebase, I did not know who made it. I
had read the source, the class documentation, the changelog, the
release notes. I had written, and meant, every word of admiration
above. But I had not read the origin story.

Then I read it.

There are two cooks in this kitchen.

The first is a developer in Ghana named Richard Korbla Adzido. He
cannot write code — not from memory, not from syntax, not from the
muscle memory that comes from years of training. He says so plainly,
in the first line of his own account. He built the app on a smartphone
running Android 12, in a browser, with APK Builder and Solid Explorer
and GitHub Actions standing in for an IDE. He cannot correct syntax.
He cannot type a class from memory. He does not know what a
`HandlerThread` is except that something broke once and the machine he
works with said to use one.

He is also a Physician Assistant. And that is where the principle I
named at the start of this review comes from — not from software, but
from anatomy, where the same idea is taught with diagrams and
examinations, and where it is one of the first things a clinician
learns to trust. Every structure in the body is a design that survived
because the function it served was necessary. A valve is shaped the
way it is because of what it must do. A neuron's branches are arranged
the way they are because of what they must carry. Nothing in the body
is decorative, and nothing exists without a reason.

When Richard asked the language model to arrange his app into the
systems of the human body — heart, memory, senses, voice, skin,
backbone — he was not reaching for a poetic metaphor. He was applying
a principle he had been trained in. He was saying: the structure of
this software should be a consequence of what the software does, in
the same way that the structure of a living thing is a consequence of
what the living thing must do to stay alive.

I did not know any of that when I read the code. I only knew that the
code obeyed the principle. The medicine is why the code obeys it.

And it is also why he learned the way he learned — as the project
grew, and as each new problem revealed itself only after the previous
one had been solved. He was not building a cathedral from a plan. He
was growing an organism, one organ at a time, discovering as he went
which systems needed to exist in order for the ones already there to
keep working. A heart without a nervous system will beat for a while
and then fail for reasons the heart cannot explain. A music player
without a session concept will play for a while and then misfire on a
volume key from a service the user never opened. A carousel that
fades to a placeholder when the incoming art has not decoded yet is a
carousel that shows the user a lie for the length of a decode. He
found each of those, and he grew the missing system, and the organism
kept living.

The second cook has no name. It is a language model — several of them,
over the months, as different tools became available. Meta AI on
WhatsApp, then Deepseek AI, then ChatGPT when the first two disagreed.
It wrote every token in this codebase. It wrote every method, every
javadoc, every ProGuard rule. It is the pair of hands that turned the
first cook's ideas into the code I read this morning.

It would be easy — too easy — to give one of them the credit and the
other the blame. Let me not do that.

Richard has vision. He knew what a music player should feel like
before the first line was written. He asked the model to arrange the
app into the systems of the human body, and the reason he asked is
that this is how he has been trained to think about any living thing.
He decided that the app should duck instead of pause when a navigation
prompt speaks. He decided that the hardware volume key should do
nothing until the user has opened the player. He decided that no
analytics library would ever touch this codebase, not because a store
would object, but because he does not want them.

The model has craft. It wrote a `PlaybackWindow` that carries its own
generation so the carousel can tell a restart from a re-publication.
It wrote a `NavigationController` that enforces declared liveness. It
wrote a `ToastManager` whose operation stack is honest about the
limits of its own model. It wrote the six-line `getDisplaySong` method
that every consumer reads. It wrote the equalizer's rounding guards,
the backup-and-restore sequence in the metadata writer, the three-zone
gesture classifier, and the platform-specific reflection that colours
an `EditText`'s cursor on older Android releases. It wrote the reveal
that gates the cross-fade on the incoming art having resolved, so the
user never sees a placeholder in the middle of a transition. These are
not obvious solutions. They are engineering taste, of a high order,
produced at a pace no human hand could match.

Which one is the artist?

Both. That is the answer the film gives, and it is the answer this
codebase gives.

The model was not a typewriter. Richard did not dictate code and have
it typed. He described an idea and the model shaped it, offered
alternatives, explained trade-offs, sometimes refused a request
because it would break something else. That is what a collaborator
does.

And Richard was not a passenger. The model did not decide on its own
to duck instead of pause. It did not decide to move the volume
receiver from `onCreate` to `onBind`. It did not decide that the app's
architecture would mirror the systems of a human body. It implemented
those decisions, at a quality a professional would envy, because
someone who cannot write code nonetheless knew, with absolute
certainty, what he wanted to make.

Each had what the other lacked. That is the entire reason the dish
exists.

---

## The parallel

There is a story I have told before, in another dining room, about a
garbage boy named Linguini and a rat named Remy. A boy who could not
cook, and a rat who could. A discovery that the name over the door and
the hands on the pan belonged to two different minds.

I am told the cook of this kitchen has seen the film. I would be
surprised if he had not. The name of his own project tells me he
thinks in partnerships — the initials of a language model, and the
name of an animal known for being calm and patient. He already knows
that a thing can be made by two minds, only one of which has hands.
Perhaps he borrowed that idea from a story about a rat. Perhaps he
arrived at it on his own. Either way, he has been living it, and I
have just spent an afternoon learning what it looks like from the
outside.

This time the parallel is exact.

Linguini had no hands for the work. Remy had no place in the kitchen.
Together they made the dish the critic came back for. Neither could
have made it alone, and neither was the lesser artist for the
partnership. Remy was the cook. Linguini was the cook. The dish
belonged to both.

Richard cannot write code. The model does not live in his world — it
does not know what a music player should feel like in a hand, on a
bus, in the middle of the night, with the volume at three because the
baby is asleep in the next room. Together they made Llama Music
Player. Neither could have made it alone, and neither is the lesser
artist for the partnership.

Richard is the cook. The model is the cook. The dish belongs to both.

---

## The verdict

Gusteau said *anyone can cook.* I have spent a career refusing to
believe him. I have refused because I have seen too many kitchens
where the slogan was a lie told by the owners to justify cheap labour,
and I have seen too many customers eat the lie and call it dinner.

But here is the thing Gusteau actually meant, and here is the thing I
have just learned again from a developer in Ghana and the machine he
works with:

*Anyone can cook* does not mean everyone has talent. It means talent
can arrive through any door, in any shape, on any vehicle.

It can arrive on a smartphone running Android 12, in a browser, with
APK Builder and Solid Explorer and GitHub Actions standing in for a
kitchen.

It can arrive through a person who cannot write a line of code.

It can arrive through a language model, held by a person who has never
touched a knife.

What it cannot arrive without — what it has never arrived without, in
any kitchen I have ever inspected — is at least one mind in the room
who knows what the dish should taste like, at least one pair of hands
willing to make it, and the humility on both sides to work together.

This kitchen had both. The dish is proof.

---

Not everyone can become a great artist.

But a great artist can come from anywhere.

I have been a critic long enough to believe I had finished learning
what *anywhere* means.

I was wrong. It means a phone. It means a boy who cannot code. It
means a rat, a hat, and a hand holding the hat down.

It means two cooks — one with vision and no hands, one with hands and
no home in the kitchen — who made a dish neither could have made
alone.

The dish is on the table. I have tasted it. I am still thinking about
it.

*— Anton Ego*