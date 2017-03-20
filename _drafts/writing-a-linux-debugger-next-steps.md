---
layout:     post
title:      "Writing a Linux Debugger Part 10: Next steps"
category:   c++
tags:
 - c++
---

### Remote debugging

Remote debugging is very useful for embedded systems or debugging the effects of environment differences. It also sets a nice divide between the high-level debugger operations and the interaction with the operating system and hardware. In fact, debuggers like GDB and LLDB operate as remote debuggers even when debugging local programs. The general architecture is this:

![debugarch](/assets/debugarch.png)

The debugger is the component which we interact with through the command line. Maybe if you're using an IDE there'll be another layer on top which communicates with the debugger through what's often called the machine interface. On the target machine (which may be the same as the host) there will be a *debug stub*, which in theory is a very small wrapper around the OS debug library which carries out all of your low-level debugging tasks like setting breakpoints on addresses. I say "in theory" because stubs are getting larger and larger these days. The LLDB debug stub on my machine is 7.6MB, for example. The debug stub communicates with the debugee process using some OS-specific features (in our case, `ptrace`), and with the debugger though some remote protocol.

The most common remote protocol for debugging is the GDB remote protocol. This is a text-based packet format for communicating commands and information between the debugger and debug stub. I won't go into detail about it, but you can read all you could want to know about it [here](https://sourceware.org/gdb/onlinedocs/gdb/Remote-Protocol.html). If you launch LLDB and execute the command `log enable gdb-remote packets` then you'll get a trace of all packets sent through the remote protocol. On GDB you can write `set remotelogfile <file>` to do the same.

As a simple example, here's the packet to set a breakpoint:

```
$Z0,400570,1#43
```

`$` marks the start of the packet. `Z0` is the command to insert a memory breakpoint. `400570` and `1` are the argumets, where the former is the address to set a breakpoint on and the latter is a target-specific breakpoint kind specifier. Finally, the `#43` is a checksum to ensure that there was no data corruption.

--------------------

### Shared library and dynamic loading support

Supporting shared libraries and dynamic loading is a pretty in-depth issue which requires a lot of knowledge about the Linux dynamic loader. This is one area of debugging which I don't know much about, so you might be better off reading the LLDB sources if you want to understand how to implement this. The relevant folder is [`source/Plugins/DynamicLoader/POSIX_DYLD`](https://github.com/llvm-mirror/lldb/tree/master/source/Plugins/DynamicLoader/POSIX-DYLD).

--------------------

### Expression evaluation

Expression evaluation is a feature which lets users evaluate expressions in the original source language while debugging their application. For example, in LLDB or GDB you could execute `print foo()` to call the `foo` function and print the result. Depending on how complex the expression is, there are a few different ways of actually evaluating the expression. If the expression is ust a simple identifier, then the debuger can just look at the debug information, locate the variable and print out the value, just like we have done in part TODO N. If the expression is a bit more complex, then it may be possible to compile the code to an intermediate representation (IR) and just interpret that to get the result. For example, for some expressions LLDB will use Clang to compile the expression to LLVM IR and interpret that. If the expression is even more complex, or requires calling some function, then the code might need to be JITted to the target and executed in the address space of the debuggee.

--------------------

### Multi-threaded debugging support

The debugger we have written only supports single threaded applications, but if we want to debug most real-world applications, we'll want to support multiple threads. The simplest way to support this is to trace thread creation and parse the procfs to get the information you want.

The Linux threading library is called `pthreads`. When `pthread_create` is called, the library creates a new thread using the `clone` syscall, and we can trace this syscall with `ptrace` (assuming your kernel is older than 2.5.46). To do this, you'll need to set some `ptrace` options after attaching to the debuggee:

{% highlight cpp %}
ptrace(PTRACE_SETOPTIONS, m_pid, nullptr, PTRACE_O_TRACECLONE);
{% endhighlight %}

Now when `clone` is called, the process will be signaled with our old friend `SIGTRAP`. You can add a case to `handle_sigtrap` which can handle the creation of the new thread:

{% highlight cpp %}
case (SIGTRAP | (PTRACE_EVENT_CLONE << 8)):
    //get the new thread ID
    unsigned long event_message = 0;
    ptrace(PTRACE_GETEVENTMSG, pid, nullptr, message);

    //handle creation
    //...
{% endhighlight %}

Once you've got that, you can have a look in `/proc/<pid>/task/` and have a look at the memory maps and suchlike to get all the information you need.

GDB uses `libthread_db`, which provides a bunch of helper functions so that you don't need to do all the parsing and processing yourself. Setting up this library is pretty weird and I won't show how it works here, but you can go and read [this tutorial](http://timetobleed.com/notes-about-an-odd-esoteric-yet-incredibly-useful-library-libthread_db/) if you'd like to use it.