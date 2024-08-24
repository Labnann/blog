---
title: "Google Summer of Code 2024: Interactive MKV"
date: 2024-08-20
description: Creation of the world's first Interactive MKV and Interactive MKV Player
---

We all know what MKV is. It's a flexible and open standard Multimedia Container Format
that we love and use to design audio, video, subtitle tracks in a single file. Making
it perfect for storing movies and TV shows with multiple audio tracks or subtitle tracks,
making it ideal for storing multiple languages.

As the technology advances, so does our perception of entertainment. And there has been
a surge of interactive multimedia. Unfortunately, there is no open standards defining
a interactive multimedia should be implemented. Making it not only hard to create for
a creative author who doesn't want to bother with lots of technological difficulties,
but also it forces a device to depend on unnecessarily complex technologies due to the
technical debt.

With these in mind, my mentor Steve Lhomme, the creator of Matroska and I went on a
journey this summer 2024 to create an open standard to support interaction over MKV 
and an MKV Player that supports this standard over our favourite VLC Media Player.

The basic idea: MKV already has a way to define "Chapters" in a video, through the
[Chapter Codec](https://datatracker.ietf.org/doc/draft-ietf-cellar-chapter-codecs/). 
These chapters are the main pillars of this creation. A chapter lets us split a video 
into several segments. We have introduced MatroskaJS scripting for the "entry" and 
"leave" portion of these chapters. Therefore: at entry, a set of choices can be 
displayed to a user for the chapter's length of time, then according to the user's 
choice selection, "leave" script is executed. At that point, author can decide to 
control the flow of the video based on the user's choice.

We explored various JavaScript Engines to implement this. The acceptance criteria was how
small the binary size is, and the license must be LGPL compatible. After exploring QuickJS, 
MuJS, DuktapeJS and Hermes, we fixated on [Duktape](https://duktape.org/). But why a  new
script variant when there is already exists MatroskaScript? 

Well, MatroskaScript defined in the current MKV specification has only 
one  command GotoAndPlay, which is insufficient for interactive behavior. Moreover, the 
specification gives a hint about the future: that it will be like ECMAScript with supports like 
commenting. Therefore, shifting the execution engine to use a ECMAScript engine makes sense.
However, an issue with ECMAScript is, the integer support; it does not hold a value properly
and starts rounding integer numbers after 56 bit, losing backwards compatibility.

Technical details regarding the project
=======================================

The project is implemented as part of the mkv module in demux, at:
`vlc-src/modules/demux/mkv`. There was already an implementation of Chapter Codec, which
included the implementation of previous GotoAndPlay command. From this we extracted
the "script" interpreter implementation, and created a MatroskaJS interpreter 
implementation related classes. Both of them are connected via the `matroska segment parser`.
The segment parser binds the right interpreter based on "ChapProcessCodecID".

The JavaScript Frontend
------------------------

Due to the nature of JavaScript Engines, we decided to create two types of functions:

- Functions named like js_execute_CommandName. These functions are pushable to the duktape api,
and user scripts have the ability to access these functions. In these functions we handle
the typechecking and pass it to:

- Functions named like execute_CommandName, these are actual implementations of the Chapter Commands
that are proposed for the new Chapter Codec implementation and can be regarded as the start of 
backend.

Handling The Choice Display
---------------------------

Using VLC Subpicture API, we created buttons for each choices.

![ChoiceButton](https://raw.githubusercontent.com/Labnann/blog/main/static/img/gsoc24-interactive-mkv-test.png)

The implementation is something like: create a button at the start
of the chapter if the user has signalled it via the CommitChoices()

command. Then, if the button states are changed, for example: if user
pressed a button, re-create parts of that button to simulate 
button press user experience.

Handling Input Events
---------------------

We forwarded VLC's mouse event with x, y coordinates to our implementations.
Using this, along with the button dimension calculation from the display stage,
we are able to figure out where button rectangles are located. 

This allowed us to create "mouse event handler". The event handler can fire 
an event, which is connected to the vlc's original forwarded mouse event.
When event handler fires the event, all subscribed "mouse operables" that
are located at the correct region update their internal states. Then we
made the buttons we created above implement the "mouse operable" interface to
connect it all together.

Security Considerations
-----------------------

- We have made sure to handle the data types that are passed into the commands
to comply with the new MatroskaJS Specification.

- The script has 3 seconds timeout, to deal with issues like forever loop
within the script.

- Custom error handlers are implemented based on Duktape's recommendations.

Related Merge Requests
=======================
These are the two merge requests containing the project:

1. https://code.videolan.org/videolan/vlc/-/merge_requests/5925

2. https://code.videolan.org/videolan/vlc/-/merge_requests/5619

Here is Steve's part of the Merge Request regarding MatroskaJS:

https://github.com/ietf-wg-cellar/matroska-specification/pull/835

Apart from this, various bugs in the contribs is patched to make the VLC building process
smoother.
