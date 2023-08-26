---
title: "Google Summer of Code Journey:  Hacking Libtool: Part 1"
date: 2023-08-15
description: Hacking Libtool - Part 1, why hack Libtool?
---

While working in VLC.js for VideoLAN in Google Summer of Code 2023, I came
accross an interesting problem. You see VLC uses GNU Autotools to 
create their compilation script, like most other C/C++ projects. The 
compilation process is simple, if all dependencies are installed. 

First you bootstrap with ./bootstrap
Then you configure, with ./configure [--enable-params] [--disable-params]
[other params]
And finally you compile, using GNU/Make, with the command 'make'.


In the configure step, among various enable/disable parameters, there
is one parameter called "shared". This allows the modules to be compiled
as dynamic libraries, making them dynamic linkable.

So, to enable dynamic linking for wasm32-unknown-emscripten I had to just
do this one thing: add --enable-shared to the configure parameters,
right?

Wrong! Turns out even if we use --enable-shared in case of wasm32-unknown-emscripten
it does static compilation! I first realized this when I tried 
--enable-vlc with --enable-shared just to see how it turns out. And
the configuration fails with the error:
"VLC is based on plugins. Shared libraries cannot be disabled."


Something is wrong! This is not supposed to happen, I have tried with 
x86_64-linux host, and it works perfectly, then why not in case of 
emscripten? For some reason, shared libraries can not be enabled!
Time for some investigation.

#### What and why wasm32-unknown-emscripten?
For that, you have to know what cross compilation triplet is. Briefly saying,
it is \<Architecture\>-\<Vendor\>-\<Operating System\>, so here wasm32 is the
architecture here, emscripten is the Operating System taken to be running
on the browser. VLC.js is basically vlc media player ported to web assembly,
running with Emscripten in browser, hence we use this.

So, the first thing I did was enable building of vlc binaries,  with 
--enable-vlc, as vlc binaries can not be built without dynamic loading 
enabled. There has to be a check there, that makes sure that shared 
modules are enabled.  Opening the configure script and searching for 
"VLC is based on plugins", we indeed find the condition there:


if test "${enable_shared}" = "no" -a "${enable_vlc}" != "no"

So, something must have caused enable_shared to be no, But what is it?

Something must have assigned enable_shared to be no, so let's search
for "enable_shared=no", and just few lines before the above line, we
find that:

test no = "$can_build_shared" && enable_shared=no

So, the value of enable_shared came from can_build_shared, so now we
search for "can_build_shared=no" to find out that its value came from
the variable dynamic_linker. And it turns out the value of dynamic_linker
is pre-assigned based on host_os. For example, if host_os is android,
dynamic linker is "Android linker" and so on. There seemed to be only
one purpose of this variable, that is to indicate whether a dynamic linker 
exists for that particular host_os. Which does not exist for the host_os
I am targetting, "emscripten".

But the name, "Android linker" for android has to come from somewhere. 
The configure script itself is generated from various configure.am files
when running the bootstrap script as mentioned earlier. So, I make a quick search:

git grep -r "Android linker"

Only to find nothing. So this value does not exist in the repository as
a configuration file or anything, rather the value came from outside the
repository. My mentor Alexandre Janniaux was very helpful to point out
that the value comes from libtool, and what we can do at this point is to
hack libtool and add dynamic_linker for emscripten at libtool/m4/libtool.m4
and use that custom libtool.

And in libtool.m4 there it was, the "Android linker" for the host 
linux*android. Taking inspiration from how android section was implemented,
we added a new host_os named emscripten, with dynamic_linker to be 
"Emscripten linker", and bingo! After rebootstrapping with the new libtool
the configuration script ran successfully without switching to static mode 
silently.

#### What is libttool?
Libtool is a generic library support script that hides the complexity of 
using shared libraries behind a consistent, portable interface. Different
compilers take different flags, commands or approaches to compile shared
libraries. Therefore compiler front end becomes inconsistent. Which in terms,
forces us to use too many unnecessary conditionals in Makefiles. To fix
this issue, Libtool was developed, it handles the differences internally
while giving a consistent front end that we can use for the Makefiles,
enabling us to compile for any other targets whenever needed.

Later, we configured emscripten host in libtool to make compilation
compatible with various features in Emscripten compiler that is different
from Clang or GCC. The details will be in part two!

In this article we learned and reviewed:
- How projects are typically compiled and built and a bit of inner
  mechanisms.
- Debugging build scripts.
- How to get started with adding a new host target to libtool.

Stay tuned for the part two! And stay safe.
