---
layout: article_page
title: "Avoiding errors in GDB with solib-absolute-prefix"
date: 2014-01-10 20:17:04
description: "A short tip on avoiding errors by using solib-absolute-prefix when running dynamically linked executables remotely with a cross targeted GDB."
tags: gdb linux
---

Here's a quick tip on a problem I ran into recently, when I decided to debug a VOIP stack on a PowerPC box with GDB from my x86 dev PC. I found a few people on the TI E2E forum struggling with the same issue, the solution was not immediately forthcoming on Google, but was found after a quick glance at the [GDB reference card](http://www.refcards.com/docs/peschr/gdb/gdb-refcard-a4.pdf).

Say you want to debug a dynamically linked executable with GDB, which is cross compiled and running on a remote target using gdbserver, you could get errors related to the loading of the shared object libraries when the program is set running. Here is the output with such errors that I got during my debug session:

<!--more-->

On the development host:

<div class="preformatted_console"><pre>
elangley@captainhook:~/voipboxdev/bsp/ltib-mpc8315erdb-20080630$ bin/gdb rootfs/nv/vapp 
GNU gdb 6.6.50.20070620-cvs
Copyright (C) 2007 Free Software Foundation, Inc.
GDB is free software, covered by the GNU General Public License, and you are
welcome to change it and/or distribute copies of it under certain conditions.
Type "show copying" to see the conditions.
There is absolutely no warranty for GDB.  Type "show warranty" for details.
This GDB was configured as "--host=i686-pc-linux-gnu --target=powerpc-linux".
For bug reporting instructions, please see:
bug-gdb@gnu.org.
..
(gdb) target remote 192.168.7.2:4444
Remote debugging using 192.168.7.2:4444
warning: Unable to find dynamic linker breakpoint function.
GDB will be unable to debug shared library initializers
and track explicitly loaded dynamic code.
0x0ffd7af8 in ?? ()
(gdb) b fprintf
Breakpoint 1 at 0x1006c0c4
(gdb) c
Continuing.
Error while mapping shared library sections:
linux-vdso32.so.1: No such file or directory.
warning: .dynamic section for "/lib/libpthread.so.0" is not at the expected address (wrong library or version mismatch?)
Error while mapping shared library sections:
/usr/lib/libsyslogr.so.0: No such file or directory.
Error while mapping shared library sections:
/usr/lib/libwatchdog.so.0: No such file or directory.
warning: .dynamic section for "/usr/lib/libncurses.so.5" is not at the expected address (wrong library or version mismatch?)
Error while mapping shared library sections:
/usr/lib/libvappif.so.0: No such file or directory.
warning: .dynamic section for "/usr/lib/libdbus-glib-1.so.2" is not at the expected address (wrong library or version mismatch?)
warning: .dynamic section for "/usr/lib/libdbus-1.so.3" is not at the expected address (wrong library or version mismatch?)
warning: .dynamic section for "/usr/lib/libgobject-2.0.so.0" is not at the expected address (wrong library or version mismatch?)
warning: .dynamic section for "/usr/lib/libgthread-2.0.so.0" is not at the expected address (wrong library or version mismatch?)
warning: .dynamic section for "/usr/lib/libglib-2.0.so.0" is not at the expected address (wrong library or version mismatch?)
warning: .dynamic section for "/lib/libc.so.6" is not at the expected address (wrong library or version mismatch?)
Error while mapping shared library sections:
/lib/ld.so.1: No such file or directory.
warning: .dynamic section for "/lib/librt.so.1" is not at the expected address (wrong library or version mismatch?)
Error while mapping shared library sections:
/usr/lib/libRCP.so.0: No such file or directory.
Error while mapping shared library sections:
/usr/lib/libffi.so.6: No such file or directory.
Breakpoint 1 at 0x470c0

Program received signal SIGSEGV, Segmentation fault.
0x48031958 in ?? ()
(gdb)
</pre></div>


And on the target:

<div class="preformatted_console"><pre>
root@voipbox:~# /gdbserver *:4444 /nv/vapp
Process /nv/vapp created; pid = 346
Listening on port 4444
Remote debugging from host 192.168.7.1
gdb: error initializing thread_db library: version mismatch between libthread_db and libpthread
gdb: error initializing thread_db library: version mismatch between libthread_db and libpthread
gdb: error initializing thread_db library: version mismatch between libthread_db and libpthread
gdb: error initializing thread_db library: version mismatch between libthread_db and libpthread
gdb: error initializing thread_db library: version mismatch between libthread_db and libpthread
gdb: error initializing thread_db library: version mismatch between libthread_db and libpthread
gdb: error initializing thread_db library: version mismatch between libthread_db and libpthread
gdb: error initializing thread_db library: version mismatch between libthread_db and libpthread
gdb: error initializing thread_db library: version mismatch between libthread_db and libpthread


========
vPort Release +D2Tech+ VPORT  VPORT_R_1_5_28 
========

               D2 Technologies
      _   _  __                           
     / | '/  /_  _  /_ _  _  /_  _  ._   _
    /_.'/_  //_'/_ / // //_///_//_///_'_\ 
                                _/        

             Voice Over IP Products

                 www.d2tech.com
</pre></div>

You can see from the output in GDB that it is trying to interpret libraries on the dev host as candidates for being dynamically linked in to the cross compiled binary for the PPC target ("warning: .dynamic section for "/usr/lib/..."), which is clearly wrong. That appeared to interfere with my debugging session somewhat, as the program ran straight past my breakpoint and seg faulted.

The solution is to set the solib-absolute-prefix, which will direct GDB to examine shared libraries built for the target, instead of those on the host system. Here I set the prefix to point to the rootfs built for the target by LTIB:

<div class="preformatted_console"><pre>
(gdb) set solib-absolute-prefix ~/voipboxdev/bsp/ltib-mpc8315erdb-20080630/rootfs
(gdb) b fprintf
Breakpoint 1 at 0x1006c0c4
(gdb) target remote 192.168.7.2:4444
Remote debugging using 192.168.7.2:4444
0x0ffd7af8 in ?? ()
   from /home/elangley/voipboxdev/bsp/ltib-mpc8315erdb-20080630/rootfs/lib/ld.so.1
(gdb) c
Continuing.

Breakpoint 1, 0x1006c0c4 in fprintf@plt ()
(gdb)
</pre></div>
And debugging commences.



Incidentally this is reported in a much clearer manner in newer versions of GDB. Above I was using GDB 6.6 included with a Freescale LTIB BSP which is a few years old. Trying the same with GDB 7.6, built by Yocto Dora, suggests an appropriate action along with the error:

<div class="preformatted_console"><pre>
elangley@captainhook:~$ ./yocto_ppc_build_area/mybuilds/tmp/sysroots/x86_64-linux/usr/bin/ppc7400-poky-linux/powerpc-poky-linux-gdb voipboxdev/bsp/ltib-mpc8315erdb-20080630/rootfs/nv/vapp
GNU gdb (GDB) 7.6
Copyright (C) 2013 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later &lt;http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "--host=x86_64-linux --target=powerpc-poky-linux".
For bug reporting instructions, please see:
&lt;http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /mnt/src/bsp/ltib-mpc8315erdb-20080630/rootfs/nv/vapp...done.
(gdb) target remote 192.168.7.2:4444
Remote debugging using 192.168.7.2:4444
warning: Unable to find dynamic linker breakpoint function.
GDB will be unable to debug shared library initializers
and track explicitly loaded dynamic code.
0x0ffd7af8 in ?? ()
(gdb) b fprintf
Breakpoint 1 at 0x1006c0c4
(gdb) c
Continuing.
warning: Could not load shared library symbols for 16 libraries, e.g. linux-vdso32.so.1.
Use the "info sharedlibrary" command to see the complete listing.
Do you need "set solib-search-path" or "set sysroot"?

Breakpoint 1, 0x1006c0c4 in fprintf@plt ()
(gdb)
</pre></div>

So there we are.

<br />
<br />


