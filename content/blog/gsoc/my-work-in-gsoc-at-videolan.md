---
title: Google Summer of Code - My work At VideoLAN
date: 2023-08-26
description: Google Summer of Code Work Details
---

My work at Google Summer of Code, to simply explain is: enabling dynamic
loading for wasm32-unknown-emscripten. To understand it, we need to delve
into how VLC's plugins are loaded.

All vlc modules are declared in the source code using the series of macro
that starts with vlc_module_begin() and ends with vlc_module_end().

For example:

```
vlc_module_begin()
    set_shortname(N_("hello"))
    set_description(N_("Hello World Module"))
    set_capability("dummy", 0)
    set_callbacks(Open, Close)
vlc_module_end()
```

In this example callbacks we define how the module is going to be initiated.
We give it name and capability (category) and how the module will be 
closed.

If we see the definition vlc_module_begin macro, we will find that
it is expanded into a function named vlc_entry which takes a callback
vlc_set and opaque as input. 

Now let us look into bank.c. In case of shared loading, in module_InitDynamic 
we find that the function is loaded from a dynamic plugin file using dlopen.
Then using dlsym, the vlc_entry function is searched and resolved.
In case of static module, there is a global list of static entries
named vlc_static_modules.

In both cases, upon finding the entry function, which we know takes a callback, 
it is passed into the vlc_plugin_describe function. From here,
the entry function is called with the callback vlc_plugin_desc_cb 
and opaque. Where opaque, in this context is a newly initiated 
vlc_plugin_t instance.

If we look into vlc_plugin_desc_cb we can see that there is a switch
statement, that switches a property and based on the property it runs
some code.

So, the entry function now has the ability to use the callback vlc_set
to invoke vlc_plugin_desc_cb and do various things like this:

vlc_set (opaque, NULL, VLC_MODULE_CREATE)

To simplify this call, there are helper functions like vlc_plugin_set,
vlc_module_set, vlc_config_set. But they all in the end invokes this.
This way the plugin describes itself and all its capabilities until
it hits the vlc_module_end ie. the return statement.

But where is the point that decides whether to load with dynamic module
or shared module? To answer this we have to look into the function
module_loadPlugins. This function itself is called from libvlc_new
at some point which is exposed to the front end. During compilation
there is a configuration variable named HAVE_DYNAMIC_PLUGINS.

My work is based on this, to get "HAVE_DYNAMIC_PLUGINS" enabled for
wasm32-emscripten. Which will cause module_loadPlugins to call
AllocateAllPlugins which recursively looking for the modules in the 
given path passed via environment variables or other methods from the 
front end.

Now to enable this I had to make changes into libtool to allow 
it inform it about how to "compile" shared modules. 

While working on this, I came accross an interesting problem. You see 
VLC uses GNU Autotools to create their compilation script, like most 
other C/C++ projects. The compilation process is simple, if all 
dependencies are installed. 

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

So what is wasm32-unknown-emscripten?
For that, you have to know what cross compilation triplet is. Briefly saying,
it is <Architecture>-<Vendor>-<Operating System>, so here wasm32 is the
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

So, what is libttool?
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
from Clang or GCC.

By now we already know where to change for emscripten host. So later is 
pretty straightforward. That is to understand what is needed for successful
compilation and configure libtool's source accordingly.
Look around the source! There are examples everywhere.
Still we need to make sure what we need and carefully enable the required 
features corrosponding to the compiler.

Well, since my shared compilation is not being forcefully disabled, 
now, I can try to figure out from the error and warning messages which 
compiler options should be removed and which are to be added.

For example, at this stage, configuring with --enable-shared and
compiling it gives the warning: 

```
emcc: warning: linking a library with `-shared` will emit a static 
object file. This is a form of emulation to support existing build 
systems.  If you want to build a runtime shared library use the 
SIDE_MODULE setting. [-Wemcc]
```
Now from the documentation of emscripten we do find out that SIDE_MODULE
is emscripten's way of compiling a code as shared module, as opposed
to -shared. So, we have to use that instead. But how do we add this
to libtool?

For this we use the LT_TAGVAR method to modify archive_cmds. This simply 
edits the part of libtool that creates dynamic library archives to answer: 
"Which parameters will I run to create the archive?"

.. m4 ::
  _LT_TAGVAR(archive_cmds, $1)='-s SIDE_MODULE=1'

 

Libtool has default way of naming each library archives and shared
objects, which include, name, relese and extension. If we want we 
can change this default behavior by changing library_name_spec and
soname_spec. 

.. m4 ::
   library_names_spec='$libname$release$shared_ext'
   soname_spec='$libname$release$shared_ext'

We can also tell libtool which command parameter is 
necessary for position independent code setting `_LT_COMPILER_PIC`. For 
our purpose, we  needed `-fPIC`. However, now, you can try inspecting 
this variable for Windows section! As we know Windows does not use 
position independent code, but uses a very different method to achieve 
shared loading. That's the beauty of libtool! With all these 
complexities taken care of here, we do not need to worry about  it in 
compile time. With this we can easily configure cross compilation for 
all platforms with a single build script!

Finally, at your build path (The path from where you run your configure 
script) there is an interesting file named `config.status`. 
This file is created when you run the configure script. Observing this 
file, you can find some interesting keys. Keys like: soname_spec,
archive_cmds, lt_prog_compiler_pic. Sounds familiar right? We just
worked with these keys above, in libtool.m4!

This file helped me a lot while debugging libtool.m4, hope it helps 
you as well ;).

With the libtool.m4 ready for emscripten compilation, we can now finally
initiate compilation. Libtool will configure itself and put necessary
flags for us. At this point everything compiled successfully, till the
compilation of binaries. Then this error shows up:

.. log ::
    error: undefined symbol: libvlc_new (referenced by 
    top-level compiled C/C++ code) 
    warning: _libvlc_new may need to be added to 
    EXPORTED_FUNCTIONS if it arrives from a system library                                 
Here I just showed the error for libvlc_new function only. But this error
occurs in case of all the functions that are exposed to the front end.

But why does it occur? Well during compilation of shared modules, some
compilers demand another flag that tells which functions to be exported
i.e. can be seen by the user. Different compiler implements it differently,
emscripten has its own way of doing this as well.

In case of  We can use -s EXPORTED_FUNCTIONS to provide the function 
names. Like this:
`emcc code.c -s EXPORTED_FUNCTIONS=_function1,_function2,_function3`

We can provide them through  a file as well:
`-s EXPORTED_FUNCTIONS=@path/to/file.sym`

The file has to be a text file. Within this: functions are provided 
like this:

```
_function1
_function2
_function3
```

Doing so will expose function1, function2, function3 from code.c.

Notice that, the actual function name is function1, but while when exporting,
we use the name _function1.

Likewise, different compiler was implemented with different standards.
Some compilers do not even take file name, some compilers do not add _
before their function names. Some compilers do not even need to export
functions.

Now libtool handles this very interestingly. It takes the symbol file
path as input with --export-symbols option. In the backend in libtool.m4
we can catch this file path in the export_symbol variable.
Then the commands saved at archive_expsym_cmds is used to compile the file.

Conventionally, this file contains function list like this:

```
function1
function2
function3
```

That looks great! But there is a big problem. Emscripten takes shared
function input prefixing with _. But conventionally we do not have that!

To solve this I took the following steps:
- Take the input file.
- Add _ before each line.
- Save it to another file.
- Use that file.

But how to do all these while assigning it to 
`_LT_TAGVAR(archive_expsym_cmds, $1)`?

Turns out _LT_TAGVAR has a cool way to instruct it to run a command 
while assigning a string. You can use it like this:

`"Your Command Here ~ Normal string here that you Intended to Keep"`

For this purpose, libtool also has its own wrapper on `sed` named 
`$SED` that can be used in the strings. During build this will be converted 
into a runnable `sed` command. `$SED` is same as `sed`, except it uses `|` in place of `/`.

So the final implementation becomes something like this:

```
_LT_TAGVAR(archive_expsym_cmds, $1)='$SED "s|^|_|" $export_symbols >$output_objdir/$soname.expsym~$CC -sSIDE_MODULE=2 -shared $libobjs $deplibs $compiler_flags -o $lib -s EXPORTED_FUNCTIONS=@$output_objdir/$soname.expsy
```
To break down: 
- We take the input file with `$export_symbols`
- We use sed to add _ before every function name: 
    `$SED "s|^|_|" $export_symbols`
- We save it to `$output_objdir/$soname.expsym"`
- We preceed it with the ~ to indicate that command part has ended, its time to take the actual string.
- In the string, we can use `$output_objdir/$soname.expsym` instead of 
`$export_symbols`!

You may ask here, we have already handled -shared in archive_cmds. So why
handle it here again? The answer is: If a file has functions to export,
Libtool will use archive_expsym_cmds, otherwise, Libtool will use 
archive_cmds.

And with this, our libtool is ready for emscripten. We just create a 
patch out of it, and instruct our emscripten  build script to always
apply our patch and compile libtool in case of shared compilation.

Also it turns out that several features of emscripten are not quite
ready yet for this purpose. For example, EM_JS is not supported
in emscripten shared modules. EM_JS is a macro that allows one to
execute JavaScript inside web assembly. Also there are issues with
linker in emscripten linker for compiling C++ sources with shared 
modules. Finally, emscripten's implementation of opendir only works
with WasmFS. WasmFS is a virtual filesystem for file operations 
provided by emscripten. WasmFS is not linked with the server, so
actual directory search is not conducted by opendir.

To mitigate this, I added a JavaScript method that mounts the plugin
directory to WasmFS. After that libvlc_new is initiated and module 
loading begins.



