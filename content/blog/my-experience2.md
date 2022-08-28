---
title: Linux Kernel Bug Fixing - My Experience
date: 2022-08-27
description: LKMP Experience 
---
For the past one year, I knew about Linux Kernel Mentorship Program
which I had been wishing to join. And this summer as my university
asks me for gaining industry level knowledge, I decided why not now?

On 8th May, I applied for the Mentorship program. "Will they take
me?" I wondered. Next two days I did not notice anything to happen.
On May 10, I went to read the emails and there I realized I have 
tasks assigned to me with the deadline being on May 18. Most of the task
description scared me and I was like, I have to do these in one week?!
If you have tough looking enemy in video games at very first mission, 
you are like "Relax! It is the first mission, it must be easy." I jumped
with this similar motivation.

As you have guessed, these tasks aren't just any other screening tasks,
they are a great learning opportunity which teaching me to boot, build,
break, test, patch the kernel as well as learning to get familiar with
the release cycle and communications. I worked a lot because to reach
my target of getting the screening tasks done properly, I had a lot to
learn in just one week.

On May 27, I got the email.
"Congratulations! We are pleased to let you know that you have been 
accepted as a mentee to the Linux kernel Bug Fixing Summer 2022 mentorship. 
The program administrator will contact you with the next steps."

And my journey as a Linux Kernel Mentee started.

Linux Kernel Bug Fixing is a Linux Kernel Mentorship Program designed
for newcomers in the open source community. I have learned so many 
aspects about computer science, lower level programming, open source
development, working with community, software engineering, software 
project management, importance of defining and maintaining core 
principles in a project in the past three months. 

One of the most important things I learned is using the docs and codebase
itself to learn to work. My mentors always encouraged the documentations
and the source code that gives relevant information. 

Some of the parts of the codebase is so
simple and fascinating. Some codes have some really funny comments.
It is really satisfying, to see the code building up from top to down
at the core level of hardware, which we normally do not see in other 
software projects. 

To try and understand this bug I used this tool named ftrace. I was
fascinated by the result seeing the internals of how so many functions
getting executed one after another just to run a simple reproducer. If
you are interested about ftrace you can learn about it here: 
https://www.kernel.org/doc/Documentation/trace/ftrace.txt

However my tool of choice was GDB and Qemu. Qemu is an emulator where
I run the kernel image and using Qemu's debugger I connect gdb to it 
and start debugging. If the error path is too mysterious for me in that
I use ftrace.

I learned, even if C is not an Object Oriented Language, it is possible
to write clean codes with it. It does not have Class, but it does
have structs. The higher level modules define and provide structs 
that have void function pointers. The lower level modules implement 
the functions and inject the pointers into the structs. This way the
Dependency Injection Principle is kept intact. The lower level module 
itself defines how it needs to be initiated, used, freed and higher
level module just calls them.

Another interesting thing here to take from this project is error
handling. If an error occurs, a negative error code is propagated 
from deep within the low level to the higher level. Here is an example:

```
struct dev* dev_create(params) 
{
 struct *dev = kmalloc(sizeof(struct dev), params.flag)
 if (!dev)
    return ERR_PTR(-ENOMEM);
// initiate things
 ...
}


int dev_create_user ()
{
 ...
 struct dev = dev_create(params);
 if (IS_ERR(dev))
    return (PTR_ERR(dev));
 ...
 //do other things.

}
```    

This way error gets propagated to the surface until it is handled.
In many cases goto is used for handling errors if it is a bit complex.
Yes, if used correctly, goto is great for clean coding error handlers
and not evil.

Programmers often insert `BUG_ON`, `WARN_ON`, `ERROR_ON`, `pr_debug` 
etc which only logs when Kernel is compiled with config like Dynamic 
Debug is turned on. These not only increases readability of the code, 
they help fuzzers to try and trigger unintended program paths to find 
out bugs automatically. The kernel has some other dynamic debugging 
tools within it too, like KASAN which is used to detect out of bounds
access and use after free, KCSAN for finding data races. Also KMSAN 
which is not in the upstream yet, for finding uninit variables.

The bugs I tried to solve are triggered by syzkaller dynamically or
static analysis tools like coverity uncovered some potential bugs.
So my task description was easy, try to fix the bugs and let the 
mentors know if any difficulties are faced. Everyday I went to Syzbot, 
look through the issues that are probably valid and choose one or two 
bugs that has reproducer and try to find out what causes the problem 
and what could be the solution to the problem. 

Here is the link to Syzbot, if you are interested:
https://syzkaller.appspot.com/upstream

And here is the Coverity link for Linux Bugs:
https://scan.coverity.com/projects/linux

Linux Kernel is a complicated project. Often It was hard to come up with
the right question to ask the mentors. So I had to self study, buy one
or two books just to learn what could be the right term to use. But 
that is not all. The thing about open source is, it is not just the 
designated mentors that help, rather everyone helps. I send a patch 
that might be solution and people working from that area comments on
my patches. I get to ask questions about why they think so and they 
are really helpful and considering enough to answer these questions. This 
is one of the most wonderful experiences to me. They do not care about 
age, race, gender, ethnicity, color or religion. The open source culture
is truly awesome.

Often my fellow mentees did some very cool discussions. Those were
very informative and fun. It was a great experience to meet, learn
and work with people from different parts of the world. Every day I
is something new to learn.

And before I could understand it, the ending of this amazing adventure
arrived. But as our mentor said, the purpose was to get us started, 
with Kernel Development and we can always keep contributing to this
wonderful project. So, this is not the ending, rather the beginning.

By now I have three patches accepted in Linux and few rejected as well.
With all of them being very informative. And the big news is Linux 6.0
is going to be released soon, with my code running in it too.

