---
title: "Session 03: Benchmarking and Debugging"
linkTitle: "03. Benchmarking and Debugging"
---

Because unikernels aim to be a more efficient method of virtualization, then we should have certain mechanisms to measure this. Of course, in addition to identifying bottlenecks, we also need a way to fix any bugs.
Thus this session aims to familiarize you with the tools and setup needed to achieve a benchmark for an application or to solve any problem encountered during development.

## Reminders

At this stage, you should be familiar with the steps of configuring, building and running any application within Unikraft and know the main parts of the architecture.

## Debugging
### Debugging in Unikraft

Contrary to popular belief, debugging a unikernel is in fact simpler than debugging a standard operating system.
Since the application and OS are linked into a single binary, debuggers can be used on the running unikernel to debug both application and OS code at the same time.
A couple of hints that should help starting:

1.  In the menu, under \"Build Options\" make sure that \"Drop unused   functions and data\" is **unselected**. 
    This prevents Unikraft from removing unused symbols from the final image and, if enabled, might hide missing dependencies during development.
2.  Use `make V=1` to see verbose output for all of the commands being executed during the build.
    If the compilation for a particular file is breaking and you would like to understand why (e.g., perhaps the include paths are wrong), you can debug things by adding the `-E` flag to the command, removing the `-o [objname]`, and redirecting the output to a file which you can then inspect.
3.  Check out the targets under \"Miscellaneous\" when typing `make help`, these may come in handy.
    For instance, `make print-vars` enables inspecting at the value of a particular variable in `Makefile.uk`.
4.  Use the individual `make clean-[libname]` targets to ensure that you're cleaning only the part of Unikraft you're working on and not all the libraries that it may depend on.
    This will speed up the build and thus the development process.
5.  Use the Linux user-space platform target for quicker and easier development and debugging.

### Using GDB

The build system always creates two image files for each selected platform: 
- one that includes debugging information and symbols (<em>.dbg</em> file extension)
- one that does not

Before using gdb, go to the menu under \"Build Options\" and select a \"Debug information level\" that is bigger than 0. We recommend 3, the highest level.

![debug information level](./images/debug_information_level.png)

Once set, save the configuration and build your images.

#### Linux
---

For the Linux user-space target simply point gdb to the resulting debug image, for example:

```properties
gdb build/app-helloworld_linuxu-x86_64.dbg
```

#### KVM
---
For KVM, you can start the guest with the kernel image that includes debugging information, or the one that does not.
We recommend creating the guest in a paused state (<em>S</em> parameter):

```properties
qemu-system-x86_64 -s -S -cpu host -enable-kvm -m 128 -nodefaults -no-acpi -display none -serial stdio -device isa-debug-exit -kernel build/app-helloworld_kvm-x86_64.dbg -append verbose
```

Note that the <em>-s</em> parameter is shorthand for -gdb tcp::1234. Now connect gdb by using the debug image with:

```properties
gdb --eval-command="target remote :1234" build/app-helloworld_kvm-x86_64.dbg
```

Unless you’re debugging early boot code (until `_libkvmplat_start32`), you’ll need to set a hardware break point:

```properties
hbreak [location]
continue
```

We’ll now need to set the right CPU architecture:

```properties
disconnect
set arch i386:x86-64:intel
```

And reconnect:
```properties
tar remote localhost:1234
```

You can now run `continue` and debug as you would normally.

#### Xen
---
For Xen the process is slightly more complicated and depends on Xen’s gdbsx tool. First you’ll need to make sure you have the tool on your system.
Here are sample instructions to do that:

```properties
[get Xen sources]
./configure
cd tools/debugger/gdbsx/ && make
```

The gdbsx tool will then be under tools/debugger.
For the actual debugging, you first need to create the guest (we recommend paused state: xl create -p), note its domain ID (xl list) and execute the debugger backend:

```properties
gdbsx -a [DOMAIN ID] 64 [PORT]
```

You can then connect gdb within a separate console and you’re ready to debug:

```properties
gdb --eval-command="target remote :[PORT]" build/helloworld_xen-x86_64.dbg
```

You should be also able to use the debugging file (build/app-helloworld_xen-x86_64.dbg) for gdb instead passing the kernel image.

## Tracepoints

### Dependencies

We provide some tools to read and export trace data that were collected with Unikraft’s tracepoint system.
The tools depend on Python3, as well as the click and tabulate modules.
You can install them by running (Debian/Ubuntu):

```properties
sudo apt-get install python3 python3-click python3-tabulate
```

### Enabling Tracing

Tracepoints are provided by <em>lib/ukdebug</em>.
To enable Unikraft to collect trace data, enable the option `CONFIG_LIBUKDEBUG_TRACEPOINTS` in your configuration (via `make menuconfig` under `Library Configuration -> ukdebug -> Enable tracepoints`).

![enable tracepoints](./images/enable_tracepoints.png)


The configuration option `CONFIG_LIBUKDEBUG_ALL_TRACEPOINTS` activates **all** existing tracepoints.
Because tracepoints may noticeably affect performance, you can alternatively enable tracepoints only for compilation units that you are interested in.

This can be done with the `Makefile.uk` of each library.

```properties
# Enable tracepoints for a whole library
LIBNAME_CFLAGS-y += -DUK_DEBUG_TRACE
LIBNAME_CXXFLAGS-y += -DUK_DEBUG_TRACE

# Alternatively, enable tracepoints of source files you are interested in
LIBNAME_FILENAME1_FLAGS-y += -DUK_DEBUG_TRACE
LIBNAME_FILENAME2_FLAGS-y += -DUK_DEBUG_TRACE
```

This can also be done by defining `UK_DEBUG_TRACE` in the head of your source files. Please make sure that `UK_DEBUG_TRACE` is defined before `<uk/trace.h>` is included:


```properties
#ifndef UK_DEBUG_TRACE
#define UK_DEBUG_TRACE
#endif

#include <uk/trace.h>
```

As soon as tracing is enabled, Unikraft will store samples of each enabled tracepoint into an internal trace buffer.
Currently this is not a circular buffer.
This means that as soon as it is full, Unikraft will stop collecting further samples.

### Reading Trace Data

Unikraft is storing trace data to an internal buffer that resides in the guest’s main memory.
You can use gdb to read and export it.
For this purpose, you will need to load uk-gdb.py helper into your gdb session.
It adds additional commands that allow you to list and store the trace data.
We recommend to automatically load the script to gdb.
For this purpose, add the following line to your `~/.gdbinit`:

```properties
source /path/to/your/build/uk-gdb.py
```

In order to collect the data, open gdb with the debug image and connect to your Unikraft instance as described in Section [Using GDB](#using-gdb):

```properties
gdb build/app-helloworld_linuxu-x86_64.dbg
```

The `.dbg` image is required because it contains offline data needed for parsing the trace buffer.

As soon as you let run your guest, samples should be stored in Unikraft’s trace buffer.
You can print them by issuing the gdb command `uk trace`:

```properties
(gdb) uk trace
```

Alternatively, you can save all trace data to disk with `uk trace save <filename>`:

```properties
(gdb) uk trace save traces.dat
```

It may make sense to connect with gdb after the guest execution has been finished (and the trace buffer got filled).
For this purpose, make sure that your hypervisor is not destroying the instance after guest shut down (on qemu, add `--no-shutdown` and `--no-reboot` parameters).

If you are seeing the error message `Error getting the trace buffer. Is tracing enabled?`, you probably did not enable tracing or Unikraft’s trace buffer is empty.
This can happen when no tracepoint was ever called.

Any saved trace file can be later processed with the `trace.py` script. In our example:

```properties
support/scripts/uk_trace/trace.py list traces.dat
```

### Creating Tracepoints

Instrumenting your code with tracepoints is done by two steps.
First, you define and register a tracepoint handler with the `UK_TRACEPOINT()` macro.
Second, you place calls to the generated handler at those places in your code where your want to trace an event:

```properties
#include <uk/trace.h>

UK_TRACEPOINT(trace_vfs_open, "\"%s\" 0x%x 0%0o", const char*, int, mode_t);

int open(const char *pathname, int flags, ...)
{
      trace_vfs_open(pathname, flags, mode);

      /* lots of cool stuff */

      return 0;
}
```

`UK_TRACEPOINT(trace_name, fmt, type1, type2, ... typeN)` generates the handler `trace_name()` (static function).
It will accept up to 7 parameters of type `type1`, `type2`, etc.
The given format string `fmt` is a printf-style format which will be used to create meaningful messages based on the collected trace parameters.
This format string is only kept in the debug image and is used by the tools to read and parse the trace data.
Unikraft’s trace buffer stores for each sample a timestamp, the name of the tracepoint, and the given parameters.

## Benchmarking

## Uniprof

## Summary

## Practical Work

### 01. Work Item 1


### 02. Work Item 2

### 03. Work Item 3

### 04. Work Item 4

## Further Reading

>>>>>>> Enhancement: Adding debug documentation part.