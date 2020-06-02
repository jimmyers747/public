# Working with Container Core Dumps

## Introduction

If a container process crashes (e.g. due to use of a bad pointer) there is a
Linux facility for it to generate a "core dump" which is a file containing an
image of the processes' memory segments containing stacks, heap, etc. This
file can then be used to get a stack backtrace for the process, as well
as being able to print values of variables, function arguments, etc.

Notes:
- Crashes normally occur as a result of a Linux signal occurring, based on
  a hardware interrupt. Typically the signals are a segmentation violation
  (SIGSEGV) or bus error (SIGBUS), both related to invalid memory access.
- Process crashes are much less common for "managed code" (e.g. Python,
  .NET) because the execution system (virtual machine) should be very
  robust, and should not allow managed (user) code to do anything that
  would result in a process crash.
- When managed code calls C/C++, the virtual machine no longer is in charge,
  so process crashes are more likely.
- Generally, language-level exception handling (e.g. try/catch) doesn't handle
  signals like SIGSEGV or SIGBUS because the language context is unknown.
  These signals can be caught in C/C++ signal handlers, but in the case
  of memory errors, it is difficult to implement any graceful recovery.
  
## Preparation

In order for the Linux kernel to generate a core file, two things are needed:

1. The process must have sufficient resources defined for the core dump file.
   The command `ulimit -a` will show various process limits, and the line
   with `core file size` is the amount of disk space that can be used for a core file.
   A value of `unlimited` is good, and if necessary, it can be set using the
   command:
   ```
   ulimit -c unlimited
   ```
2. There must be a valid specification of where the core file should be written, including
   its name. This is done using the pseudo-file named `/proc/sys/kernel/core_pattern`.
   Detailed syntax is [here](https://linux.die.net/man/5/core),
   or do `man -s5 core`, but in general, you need to pick a location
   that is writable and has enough space. Within an RDU301 docker container, a good
   location is `/cntr_health` since it is accessible on the host system.  However,
   you cannot write /proc within a container, so it must be done on the host.
   Use the following tool commands:
   ```
   mkdir -p /cntr_health
   echo "/cntr_health/%e.%p" > /proc/sys/kernel/core_pattern
   ```
   The first command is just a precaution in case a process running on the host
   crashes (not strictly necessary).

## Analyzing a core file

After preparing for a core dump, and reproducing the problem, you should see a core
file in `/cntr_health` within the container, or `/var/cntr_health/<service id>` on
the host side.

To analyze the core file you need:

- A build host that contains the build toolchain, and the `gdb` program (which normally
  comes with the gcc toolchain).
- The application executable file and all shared libraries. This can be gotten
  from the RDU301 host using the command:
  ```
  docker export <container-name> >image.tar
  ```
  or the it can be extracted from the container image file.  We will assume the
  second method below.

For the example below, we assume:
- the core file is named `dotnet.1`, meaning that the executable file was `dotnet`
  and the PID was 1.
- the container image name is `rdumanagerservices-202005131.tar`

Both files are in the current working directory.

Note that `gdb` assumes any shared libraries are located where they were when the
process crashed. Since we are analyzing the core file on different machine than where
it crashed, this is not the case. So we have to:

1. Extract both the executable file and the shared libraries from the docker image
2. Tell `gdb` where the shared libraries are.

The docker image contains a number of file system layers. In the following sequence
of commands, we extract them with the goal of creating a directory called `root`
which represents the root (/) directory of the container.
```
IMAGE_TAR=rdumanagerservices-202005131.tar
mkdir root
tar xf $IMAGE_TAR -C root
for d in $(ls root/*/layer.tar); do tar xf $d -C root;done
```
Instead of doing this, we could have just copied the output of the
`docker export` command referred to above, into the `root` directory.

If we don't know where the executable `dotnet` is located, we can find it...
```
find root -name dotnet
```
and will see it is `root/usr/bin/dotnet`.

To analyze a core file `gdb` is invoked as follows `gdb <executable> <core-file>`
```
gdb root/usr/bin/dotnet dotnet.1
```
The `gdb` program will have issues with shared libraries, so we tell it the
location of the system root:
```
(gdb) set sysroot root
```
Now `gdb` should be able to read the shared libraries. And if the application was
built with debug enabled, we will be able to get a traceback with file names and
line numbers.
```
(gdb) backtrace
#0  0x00007fc9ac226f4e in rxHandler (pParams1=0x7fc4909adcb0) at src/gbp_nl.cpp:1992
#1  0x00007fc9ac22450c in gbp_nlExec (pParams=0x7fc4909adcb0) at src/gbp_nl.cpp:403
#2  0x00007fc9ac222d69 in alExecNormal (pParams=0x7fc4902747a0) at src/gbp_al.cpp:3105
#3  0x00007fc9ac22059c in gbp_alExec (pParams=0x7fc4902747a0, pNext=0x7fc9ac43c74c <waitInterval>) at src/gbp_al.cpp:390
#4  0x00007fc9ac21f96d in Run () at src/GBP.cpp:410
#5  0x00007fc95083f9f3 in ?? ()
#6  0x0000000001c51e2c in ?? ()
#7  0x00007fc9c480b9e8 in vtable for InlinedCallFrame () from ./usr/share/dotnet/shared/Microsoft.NETCore.App/2.1.18/libcoreclr.so
#8  0x00007fc9af6ddcb0 in ?? ()
#9  0x00007fc94e324858 in ?? ()
#10 0x00007fc94e324858 in ?? ()
#11 0x00007fc9af6dd4c0 in ?? ()
#12 0x00007fc95083f9f3 in ?? ()
#13 0x00007fc9af6dd530 in ?? ()
#14 0x00007fc94e324858 in ?? ()
#15 0x00007fc6047ff408 in ?? ()
#16 0x0000000000000000 in ?? ()
(gbd) 
```
At this point, we can use all of the features of `gdb`.  A few common commands are:
```
info args        # Get a list of function arguments and their values
info locals      # Get a list of local variables and their values
frame 3          # Go to a frame in the backtrace, in this case the frame for gbp_alExec().
                 # This allows us to examine args and parameters at the point.
print <expr>     # Print a variable, using a c-like syntax, e.g. "print pParams1->stCfg.idUnit"
```
