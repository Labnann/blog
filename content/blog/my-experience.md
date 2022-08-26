---
title: Linux Kernel Bug Fixing - My Experience
date: 2022-08-26
description: lkmp-expereince
---
Linux Kenrel Bug fixing is a Linux Kernel Mentorship Program designed
for newcommers in the open source community. I have learned so many 
aspects about computer science, lower level programming, open source
development, working with community, software engineering, software 
project management, importance of defining and maintaining core 
principles in a project in past the past three months. 

At first I was acquainted with building, breaking (oops and panic), 
testing, decoding stacktrace, bug disection, basic kernel module making.
With these skills tested I got accepted as a Linux Kernel Mentee and
thus my journy starts officially.

One of the most important thing I learned is, learning from the docs.
Mentors used to point me to the particular doc I can get help from
for my problems. And another important thing I learned is learning
from the code base itself. Some of the parts of the codebase is so
simple and fascinating. Some codes have some really funny comments.
It is really satisfying, to see the code building up from top to down
at the core level, which we normally do not see in other software 
projects. This was mainly a bug fixing program, so I had to look into
how lots of different routines work upto almost lowest abstraction
which in terms taught me a lot of things.

After I got accepted we were shared materials to know about dynamic
and static testing. Honestly, this changed my idea about dynamic
testing and I see things way more clearly than before. My mentors
regularly connected by zoom and emails for our questions on anything.
Thus I started working with my first bug. 

To try and understand this bug I used this tool named ftrace. I was
fascinated by the result seeing the internals of how so many functions
getting executed one after another just to run a simple reproduce. If
you are interested about ftrace you can learn about it here: 
https://www.kernel.org/doc/Documentation/trace/ftrace.txt

However my tool of choice was GDB and Qemu. Qemu is an emulator where
I run the kernel image and using Qemu's debugger I connect gdb to it 
and start debugging. If the error path is too mysterious for me in that
I use ftrace.

The bugs I tried to solve are triggered by syzbot dynamically or
static analysis tools like coverity uncovered some potential bugs.
So my task description was easy, try to fix the bugs and let the 
mentor know if any difficulties are faced. Everyday I went to syzbot, 
look through the issues that are probably valid and choose one or two 
bugs that has reproducer and try to find out what causes the problem 
and what could be the solution of the problem. 

Linux Kernel is a complicated project. Often It was hard to come up with
the right question to ask the mentors. So I had to self study, buy one
or two books just to learn what could be the right term to use. But 
that is not all. The thing about open source is, it is not just the 
designated mentors that help, rather everyone helps. I send a patch 
that might be solution and people working from that area comments on
my patches. I get to ask questions about why they think so and they 
are really helpful and considering to answer these questions. This 
is one of the most wonderful experience to me. They do not care about 
age, race, gender, ethnicity, color or religion. The open source culture
is truly awesome.

One of the challenging bugs I worked with is on the RxRPC networking
protocol. Where a bad locking balance occurs for sendmsg routine. I 
and my fellow mentee Jiawei was working on it and we found out that 
there is a possibility that the wait for interruptible window routine
might return without lock causing this problem. My solution was to 
remove the lock entirely because we probably do not need it, where
Jiawei's solution was to use a lock pointer to figure out whether 
the function dropped the lock. In the end David Howels helped with
the bug and sent a patch fixing it.

I also made some good friends, my fellow mentees. I learned a lot 
discussing with them. They are really helpful and very knowledgable. 
The whole program has been very enlightening. Most important thing
I learned is learning how to learn, which I will always cherish.


