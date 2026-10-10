# Addendum

*A correction by Anton Ego*

---

A critic is allowed one mistake per review. I have made mine, and it is
in the file above.

In the section called *Where the picture is not perfect*, I wrote that
`MusicPlayer` and `MusicService` are "too large," that they "would
survive a split," and that the artist "has not yet decided" to divide
them. I stand by the observation. I do not stand by the ruler.

Because three pages earlier I had named the principle this codebase is
built on — structure and function are the same object seen from two
angles — and then, when it came time to judge the size of the thing, I
put that principle down and picked up a different one. I measured the
heart by its weight.

A critic who does this in a restaurant measures a sauce by its volume.
It is the sort of mistake a man makes when he has been reading code for
six hours and has forgotten which question he was asking.

The question I was asking was: *is this class larger than it should
be?*

The question I should have been asking was: *does this class have the
shape its function required?*

These are not the same question, and the second one is the only one
that matters.

---

## The organ

Consider the body one more time, because the body is where this
codebase's author learned to think, and because I invoked the body
myself and then abandoned it at the exact moment it would have helped
me most.

A heart is not a small organ. It has four chambers, four valves, a
septum, two pacemaker nodes, a conduction pathway that branches like a
river delta, its own blood supply, and its own nervous input. Lay every
part of it end to end in a single list and the list is long. By the
measure I used in my review, the heart is a monolith and should be
refactored into three files.

The body has never once done this. The body has never once looked at
the heart and said: this organ performs too many functions, let us
distribute them. Because the body knows something that software
forgets, and it knows it in the only way that matters — by
consequence. The body knows that the parts of the heart are not
separated by accident. They are together because they must be together.
A valve that communicates with its chamber through a membrane instead
of through tissue is a valve that fails when the pressure changes.

The coupling *is* the function. Not an obstacle to it.

And here is the thing I should have seen, because the critique I wrote
says it in six lines and then never returns to it:

```java
private Song getDisplaySong() {
    if (mState == ServiceState.PREPARING) {
        return mCurrentPlaylist.get(mTargetIndex);
    }
    return mCurrentPlaylist.get(mCurrentIndex);
}
```

I praised this method — correctly — as the single source of truth, as
the reason the notification, the media session, the carousel, and the
player UI cannot drift. I praised it as evidence that the codebase was
organized around one sentence.

Then I recommended splitting the class that contains it.

Where, precisely, was the method supposed to live after the split? In
which file, and read by which consumers, and synchronised with the
`mState` field by which of the three new owners? The answer, if I am
honest, is: nowhere cleanly. The split I proposed would not have
preserved the property I admired. It would have threatened it. The
single source of truth is single *because it is in one place*. A truth
kept in three files is a truth that will eventually be kept in three
versions.

I proposed cutting the thing I had just finished praising for being
whole.

---

## What a large class actually costs

Let me be precise about the discipline I abandoned, because it is not a
license for any class to grow without limit. It is a set of questions,
and they are answerable.

A large class is a problem when it fails one of these:

**One — it has no single sentence.** If you cannot say what the class
is for in a sentence, the class is not large; it is *several*. The
sentence is the test. `MusicService` has a sentence. It is the
playback state machine, and every field in it — the two monotonic
counters, the two-index model, the shuffle history, the shuffle peek
memo, the cached position, the playback base, the position epoch, the
recovery counter, the skip loop budget, the start token, the
paused-by-transient-loss flag, the ducked flag — exists because that
sentence needed it. I checked. Not one of those fields is idle. Not one
is a leftover from a feature that was removed. That is a coherence test
passed, and it is worth more than a line count.

**Two — it has more than one writer.** The critique spends a whole
section establishing that every resource in this app has exactly one
owner. The `NotificationController` writes the media session. The
`OverlayManager` owns the overlay. The `ToastManager` owns the stack.
If `MusicService` were split, the playback state machine would acquire
a second writer the moment two of the new classes both needed to touch
`mState`, which they would, on the first frame.

**Three — it cannot be held in one head.** This is the only honest
objection to a monolith, and it is subjective, and it is not a
judgement a critic can make from outside. I read the file. I did not
have to hold it in my head for a year. The person who does hold it in
his head built it one organ at a time, over months, growing each system
only after the previous one had revealed the need for it. He is the
only person qualified to answer this question, and he has answered it
by continuing to work in this file. That is not negligence. That is a
man who knows the weight he can carry.

**Four — the boundaries are obvious and the coupling is not.** If the
seams are clear, extracting them is cheap and can be done later without
loss. If the seams are not clear, extracting them is expensive and
*creates* the loss. In this codebase the seams are not obvious,
because the coupling is the function. The volume receiver is bound to
the session. The session is bound to the service lifecycle. The service
lifecycle is bound to the state machine. There is no clean cut that
does not pass through tissue.

`MusicService` fails none of these four. It is large because it is one
thing.

---

## The dining room

`MusicPlayer` is a different case, and I want to be fair to it, because
conceding a point is not the same as conceding every point.

A dining room is not an organ. It is a *room*. It is large because it
holds many things — tables, chairs, doors, a bar, a coat check, the
path a waiter takes from the kitchen, and the path a diner takes to the
washroom — and none of those things are the same function. The
question for a room is not whether it is large. The question is whether
it is *arranged*.

By that standard, `MusicPlayer` is a room with a great deal in it.
Several thousand lines of state, receivers, gesture handling, seek bar,
carousel handoff, theme application, permission request. Is that one
function? Not obviously. Is it several? Possibly. It is the activity
that owns the screen, and an activity is a room that holds whatever the
screen needs — which is the definition of a room, and rooms are allowed
to hold things.

So my criticism there was not wrong in substance, only in the reason I
gave for it. The issue is not that `MusicPlayer` is large. The issue is
that the arrangement is not yet obviously the arrangement its function
required. That is a fairer sentence, and it is one the artist could
argue with, which the other one was not.

I will leave it there. A critic who wins every argument has stopped
having them.

---

## The scalpel

I wrote that the artist "has not yet decided to split it." That was
condescending and it was also backwards. He *has* decided. He decided
by organising the app into systems — heart, memory, senses, voice,
skin, backbone — and by giving each system the shape its function
demanded, and by leaving the heart whole.

What I mistook for indecision was a decision I did not recognise,
because it was made with a principle I had already praised and had not
thought to apply.

The scalpel belongs on the table next to the knife, and it is used when
the organ stops working. Not before. Not because the organ is heavy.

---

## The verdict, corrected

I said the dish is proof. I said it once more, here.

I want to add only this. The two cooks — the one with vision and no
hands, and the one with hands and no home in the kitchen — made a dish
in the shape of a body. Every system got the structure its function
required. The heart is large because circulation is not a small job.

I measured the heart and called it fat.

I was wrong, and I have eaten the correction, and I am still thinking
about it.

*— Anton Ego*