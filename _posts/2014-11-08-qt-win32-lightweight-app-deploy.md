---
layout: article_page
title: "Lightweight Qt 5 Windows build for app deployment"
date: 2014-11-08 12:41:55
description: "Build Qt for Win32 with MinGW 4.8. Reduce Qt footprint in Windows. Simplest recipe for deploying Qt applications on Windows by creating a static Qt build."
tags: qt windows
---

So you've written a Qt application, and naturally, you want people to use it. Well then you'd better prepare a build of it for Windows if you haven't already, because like it or not, that's what most of your likely users are going to be running (Unless its an OS specific tool).

Surprisingly, the Qt project doesn't provide any redistributable library packages specifically for the purpose of providing alongside a built application. The obvious method of copying DLLs from a Windows Qt installation can create a rather large footprint. The best course is to build a set of static Qt libraries optimised for size, and without dependencies on those huge ICU DLLs. So here I present to you the simplest "recipe" for distributing your Qt Windows application in a trim manner.

<!--more-->

Note that by simple I don't mean short, I will provide extra information to help you understand the process, as your build requirements may well be different to mine. Plus, other Qt building guides I found out there on the web tend to do things like instruct the reader to install the DirectX SDK, then configure a Qt build which doesn't even use it. Actually understanding whats going on helps avoid such unneeded steps, and with such understanding you may well be able to navigate your own build errors (I mention some of mine at the end).

## Aim

The plan is to build static Qt libraries, which are then statically linked into your application binary. This allows the linker the most opportunity to reduce the footprint of the final binary. A useful side effect of this tactic is that the build of Qt can be completely tailored to contain what your application requires of it. So there are some choices to make:

### Toolchain

For simplicity's sake, building will take place on a Windows installation. In theory the version of Windows doesn't matter, I used Windows XP in a VM because it is lighter weight than more recent versions.

You will need to choose which compiler you're going to use. The main choices are Microsoft Visual C++, or MinGW (Windows hosted GCC for Windows target). I installed both the MSVC2010 and MinGW4.8 builds of Qt 5.1.0, then built my application against both, and ran it on a few different flavours of Windows. On some systems, the MSVC build sometimes just wouldn't do anything when double clicking the EXE. I mean apart from a bit of disk chatter, nothing shows up on screen, no activity in the process monitor, nothing. The MinGW build always ran ok. Based on this highly scientific evidence, and a growing weariness of the reliability/absence of the MS Visual C++ 2010 Redistributable on any given system, I settled on building with MinGW 4.8.

So the instructions that follow are for setting up and building with MinGW. For MSVC, the instructions would be a bit different, but most of the supporting information still holds true.

### Graphics

Should Qt ask Windows to render any QGLWidget objects or QtQuick 2.0 code using OpenGL 2.0 calls, or use the ANGLE library to translate those OpenGL calls to DirectX API calls first? The only issue with OpenGL on Windows, as far as end-users are concerned, is that there is a chance they may still have the stock Windows OpenGL DLLs in use, which only support OpenGL 1.1. Calling into DirectX could avoid this issue. Use this non-diagrammatical flow-question-list to decide:

1. Does the application use QGLWidget with custom shaders, or QtQuick 2.0?
No: build for OpenGL, worst case stock OpenGL 1.1 DLLs on users system will suffice.
Yes: go to 2
2. Does the application need to run on Windows XP or older, or in a virtual machine?
Yes: go to 3
No: go to 4
3. Does the application actually use any 3D functionality/custom shaders?
Yes: build for OpenGL, hope/mandate that end user has a graphics card with drivers that provide OpenGL 2.0 or higher implementation.
No: build for OpenGL, provide the OpenGL32.dll from the [MSYS2 Mesa package](http://sourceforge.net/projects/msys2/files/REPOS/MINGW/i686/mingw-w64-i686-mesa-10.0.2-1-any.pkg.tar.xz/download) with your application.
4. Build with ANGLE

Number 3 relates to the assertion in the Qt documentation that ANGLE doesn't work on Windows XP. I haven't tested ANGLE on XP myself. For VMs, I have found the Mesa DLL approach to work very well in being able to run QtQuick 2.0 applications in guest systems. With the virtualised 3D acceleration enabled, the guest graphics drivers for both VMware and VirtualBox only offer OpenGL 1.4 last time I tried. As long as the application isn't too GPU intensive, the Mesa DLL provides OpenGL 2.0 support in software.

For my application, I'd made it easy on myself and could answer no at question 1.

### Get WebKitted out

If your application is using any of the classes which rely on the QtWebkit module, then obviously you'll want to build Webkit in. WebKit relies on ICU for unicode support, which means you can't chop out ICU from the build, which I have done in the example below. The reason for leaving it out of the build is because the DLLs are quite large. If ICU is needed, the best bet is to build it yourself to get a smaller icudt51.dll or use the DLL binaries which mcallegari79 has posted [here](http://qt-project.org/forums/viewthread/38489).

### OpenSSL

If your application uses the Qt Networking classes to make HTTPS connections, then it will be using OpenSSL. On windows at least, the Qt library appears to open the relevant OpenSSL DLLs during runtime, because if they are missing there is no DLL link error, and they aren't shown in Dependency Walker. Instead, HTTPS connections just won't work.

Builds of OpenSSL for windows are available [here](https://slproweb.com/products/Win32OpenSSL.html), however they are built with MSVC, so will have the same issues with relying on a good version of the Visual C++ redistributable being present on any given users system. Plus, its always good to have all your binaries built by the same toolchain.

It only takes a few extra minutes to build OpenSSL with MinGW, so if it is needed I would recommend doing that.


## Ready?

Lets keep the number of steps required down low, and the explanation on medium:

### Download stuff

* [Qt installer](http://qt-project.org/downloads) - Get the "Qt Online Installer for Windows"
* [Qt source code](http://download.qt-project.org/official_releases/qt/5.1/5.1.0/single/qt-everywhere-opensource-src-5.1.0.zip)
* [ActivePerl Community Edition](http://www.activestate.com/activeperl/downloads) - Choose the 5.16.x.x version
* [Python](https://www.python.org/download/) - I went for the "Python 2.7.8 Windows Installer (Windows binary — does not include source)"

ANGLE build only:

* [DirectX June 2010 SDK](http://www.microsoft.com/en-us/download/details.aspx?id=6812)

Webkit build only:

* [Ruby](http://rubyinstaller.org/downloads/) - I went for the "Ruby 1.9.3-p545" RubyInstaller package

Building OpenSSL only:

* [MSYS shell](http://sourceforge.net/projects/mingw/files/MSYS/Base/msys-core/msys-1.0.11/MSYS-1.0.11.exe/download)
* [OpenSSL source code](http://www.openssl.org/source/) - Choose the latest, openssl-1.0.1j.tar.gz as I write this

### Install Qt

Run the Qt installer, and choose to install the MinGW 4.8 build of whichever Qt version you are interested in using. I used Qt 5.1.0. If you are developing in Windows, the chances are you've already done this. If not, I would recommend installing Qt, building your application in Windows and testing it, before moving on to building against your self-built Qt. 

This will also give you the MinGW 4.8 toolchain, installed by default under C:\Qt\. If you choose to install Qt under a different path, alter any later instructions referring to that path. The same goes for any other paths given in the rest of this guide.

### OpenSSL build only: Install MSYS

In the post install normalization process which appears in the command prompt window, say yes to MinGW installed, at: C:/Qt/Tools/mingw48_32
Run the MSYS command prompt, when running mount, should then see an entry like:

<div class="preformatted_console"><pre>
C:\Qt\Tools\mingw48_32 on /mingw type user (binmode)
</pre></div>

Then ensure that /mingw/bin is then in the path, and gcc is the one from the MinGW toolchain that Qt installed:

<div class="preformatted_console"><pre>
$ echo $PATH
.:/usr/local/bin:/mingw/bin:/bin:/c/Perl/site/bin:/c/Perl/bin:/c/WINDOWS/system3
2:/c/WINDOWS:/c/WINDOWS/System32/Wbem

$ gcc -v
Using built-in specs.
COLLECT_GCC=C:\Qt\Tools\mingw48_32\bin\gcc.exe
COLLECT_LTO_WRAPPER=c:/qt/tools/mingw48_32/bin/../libexec/gcc/i686-w64-mingw32/4
.8.0/lto-wrapper.exe
Target: i686-w64-mingw32
Configured with: ../../../src/gcc-4.8.0/configure --host=i686-w64-mingw32 --buil
d=i686-w64-mingw32 --target=i686-w64-mingw32 --prefix=/mingw32 --with-sysroot=/t
emp/x32-480-posix-dwarf-r2/mingw32 --enable-shared --enable-static --disable-mul
tilib --enable-languages=c,c++,fortran,lto --enable-libstdcxx-time=yes --enable-
threads=posix --enable-libgomp --enable-lto --enable-graphite --enable-checking=
release --enable-fully-dynamic-string --enable-version-specific-runtime-libs --d
isable-sjlj-exceptions --with-dwarf2 --disable-isl-version-check --disable-cloog
-version-check --disable-libstdcxx-pch --disable-libstdcxx-debug --disable-boots
trap --disable-rpath --disable-win32-registry --disable-nls --disable-werror --d
isable-symvers --with-gnu-as --with-gnu-ld --with-arch=i686 --with-tune=generic
--with-host-libstdcxx='-static -lstdc++' --with-libiconv --with-system-zlib --wi
th-gmp=/temp/mingw-prereq/i686-w64-mingw32-static --with-mpfr=/temp/mingw-prereq
/i686-w64-mingw32-static --with-mpc=/temp/mingw-prereq/i686-w64-mingw32-static -
-with-isl=/temp/mingw-prereq/i686-w64-mingw32-static --with-cloog=/temp/mingw-pr
ereq/i686-w64-mingw32-static --enable-cloog-backend=isl --with-pkgversion='rev2,
 Built by MinGW-builds project' --with-bugurl=http://sourceforge.net/projects/mi
ngwbuilds/ CFLAGS='-O2 -pipe -I/temp/x32-480-posix-dwarf-r2/libs/include -I/temp
/mingw-prereq/x32-zlib/include -I/temp/mingw-prereq/i686-w64-mingw32-static/incl
ude' CXXFLAGS='-O2 -pipe -I/temp/x32-480-posix-dwarf-r2/libs/include -I/temp/min
gw-prereq/x32-zlib/include -I/temp/mingw-prereq/i686-w64-mingw32-static/include'
 CPPFLAGS= LDFLAGS='-pipe -L/temp/x32-480-posix-dwarf-r2/libs/lib -L/temp/mingw-
prereq/x32-zlib/lib -L/temp/mingw-prereq/i686-w64-mingw32-static/lib -L/temp/x32
-480-posix-dwarf-r2/mingw32/opt/lib'
Thread model: posix
gcc version 4.8.0 (rev2, Built by MinGW-builds project)
</pre></div>

### Install Active Perl

In the installation wizard, tick the box to add Perl to the path environment variable.

### Install Python

Accept the defaults. The Python install directory will be added to the path environment variable.

### Webkit build only: Install Ruby

Accept the defaults. The Ruby install directory will be added to the path environment variable.

### ANGLE build only: Install DirectX SDK

Accept the defaults. If you see the S1023 error, uninstall whatever version of the Visual C++ 2010 Redistributable you have installed, through Control Panel->Add/remove programs, as explained [here](http://support.microsoft.com/kb/2728613).

## Build stuff

### Webkit build only: Build ICU

See the Qt wiki article [here](http://qt-project.org/wiki/Compiling-ICU-with-MinGW) to build ICU with MinGW, or the follow the link in the pre-amble above to get the smaller DLLs.

### Optional: Build OpenSSL

Open an MSYS terminal, in it run the following commands:

<div class="preformatted_console"><pre>
cd /c/qtbuild
tar xzf openssl-1.0.1j.tar.gz
cd openssl-1.0.1j
./Configure --prefix=$PWD/dist no-idea no-mdc2 no-rc5 shared mingw
make depend && make && make install
</pre></div>

Once that has finished, the DLL files that you need to distribute with your Qt binary, libeay32.dll and ssleay32.dll, are in c:\qtbuild\openssl-1.0.1j\dist\bin. The include files and import libraries for Qt to build against are in dist\include and dist\lib respectively.

#### Unimportant Information Alert

Anyone arriving fresh from *nix land might look in the dist\lib directory, see .a files in there, and be thinking "Erm, I thought I was building shared libraries?". Well unlike linking against a shared object library with GCC+Binutils, where all you need at build time is the .so file, on Windows, to use a DLL, at build time the linker needs an "import library", that goes with the DLL. In Microsofts toolchain, the import library is a .lib file.

When MinGW builds a Windows DLL, its still a Windows DLL, so the linker still needs an import library in order to link more code to it. The import library gets given the .a extension in the Binutils style, hence the confusion.

### Build Qt

Extract the Qt source code zip file to, say, C:\qtbuild\qt-everywhere-opensource-src-5.1.0\.

To keep the size of the statically linked application binary down to a minimum, let's set some compile and link options to put each function or class method into a seperate section during compilation of the libraries, and then during linking, leave out unused sections. This effectively leaves out any uncalled methods, which the linker needs to figure out from main() onwards, hence this size optimisation can only really be done usefully when linking statically.

In the file qt-everywhere-opensource-src-5.1.0\qtbase\mkspecs\win32-g++\qmake.conf, edit these variables as follows:

	QMAKE_CFLAGS_RELEASE    = -Os -fdata-sections -ffunction-sections
	
	QMAKE_LFLAGS_RELEASE    = -Wl,-s,--gc-sections

As well as building the Qt libraries themselves with these options, they will be baked into the qmake.exe, which will create makefiles for your application with these options as well.

Open a fresh Windows command prompt (No MSYS shell needed for this bit), to ensure your `%path%` contains everything installed so far. Then run the following commands, depending on your build preferences.

For OpenGL support:

<div class="preformatted_console"><pre>
set path=C:\Qt\Tools\mingw48_32\bin;%path%
cd \qtbuild\qt-everywhere-opensource-src-5.1.0

configure.bat -static -opensource -release -prefix C:\Qt\5.1.0\mingw48_32_static -nomake examples -nomake demos -nomake tests -platform win32-g++ -c++11 -opengl desktop -openssl -I C:\qtbuild\openssl-1.0.1j\dist\include -L C:\qtbuild\openssl-1.0.1j\dist\lib -no-icu -skip qtwebkit
</pre></div>

For ANGLE support:

<div class="preformatted_console"><pre>
"\DirectX_SDK\Utilities\bin\dx_setenv.cmd"
set path=C:\Qt\Tools\mingw48_32\bin;%path%
cd \qtbuild\qt-everywhere-opensource-src-5.1.0

configure.bat -static -opensource -release -prefix C:\Qt\5.1.0\mingw48_32_static -nomake examples -nomake demos -nomake tests -platform win32-g++ -c++11 -opengl desktop -angle -openssl -I C:\qtbuild\openssl-1.0.1j\dist\include -L C:\qtbuild\openssl-1.0.1j\dist\lib -no-icu -skip qtwebkit
</pre></div>

The arguments to the configure batch file are the important bit in all of this. If you want Webkit, leave off the `-no-icu` and `-skip qtwebkit` options. If you aren't concerned with OpenSSL, leave off the `-openssl` options. Run `configure.bat --help` to see all the other options and tweak the build to your liking.

When that finishes, check the summary of the configuration to make sure all the settings are as required, then:

<div class="preformatted_console"><pre>
mingw32-make
mingw32-make install
</pre></div>

After installing, you should find a set of static Qt libraries, include files, Qt build binaries (qmake.exe etc) and what have you, all under C:\Qt\5.1.0\mingw48_32_static\.

## Done!

With the new Qt build ready, you can now build your application against it, using C:\Qt\5.1.0\mingw48_32_static\bin\qmake.exe at the command line as usual. Or you could add the new build of Qt to QtCreator which would have been installed in the earlier steps, and build your application with that.

You should see that the statically linked EXE created, with the minimum needed MinGW and OpenSSL DLLs, has a lot smaller footprint than the smaller dynamically linked EXE, with all the ICU and Qt DLLs that that needs to run.

<table id="captionedpicture">
	<tr><td>
		<img src="{{ site.url }}/img/blog/qt-win32-deploy/qtapp-dist-large-footprint.jpg" alt="Larger application files set, the total size is 54.5MB" />
	</td></tr>
	<tr><td>
		<img src="{{ site.url }}/img/blog/qt-win32-deploy/qtapp-dist-small-footprint.jpg" alt="Smaller application files set, the total size is 17.8MB" />
	</td></tr>
	<tr><td>Top: The first attempt at a distribution of my application, with dynamically linked EXE, QML files, the required Qt5, ICU, QJson and OpenSSL DLLs, making a rather large package.<br />Bottom: The finished result after a statically building the libraries (Except OpenSSL) and building the QML into the EXE.</td></tr>
</table>

Well, that's that. If your shiny new build of Qt is installed and you've built your application against it, then go ahead and skip on down to the comments to let everyone know how pleased you are. 

And lastly, bear in mind that although this article is based around a Windows build, the same optimisations and custom build technique could be used to reduce the footprint of a Qt application on say, an Embedded Linux device.

## Not quite done?: Build issues

As Shakespeare wrote, "The course of successful compilation never runs smooth". Or something like that. If an error brings your building of Qt5 to a screeching halt, maybe it looks like one of these below. Note that the fixes on offer here are either rather obvious, or not intended as actual fixes but will at least get the build going again and let you carry on with your day.

If you get this:

<div class="preformatted_console"><pre>
Generating Makefiles...
execute: Unknown error
   (C:/qtbuild/qt_5.1.0_src/qtbase/qtbase.pro)
   (-o)
   (C:/qtbuild/qt_5.1.0_src/qtbase)
Qmake failed, return code -1
</pre></div>

Then don't, as I initially did, try and use a copy of a git repository checked out the the v5.1.0 tag and copied into Windows from Linux, as that clearly didn't work for some reason.

If you get an error like this:

<div class="preformatted_console"><pre>
g++ -Wl,-s -Wl,-subsystem,console -o ..\..\..\bin\moc.exe .obj/release_static/mo
c.o .obj/release_static/preprocessor.o .obj/release_static/generator.o .obj/rele
ase_static/parser.o .obj/release_static/token.o .obj/release_static/main.o  -L"C
:\Program Files\Microsoft DirectX SDK (June 2010)\Lib\x86" -LC:/qtbuild/openssl-
1.0.1j/dist/lib -LC:/qtbuild/qt-everywhere-opensource-src-5.1.0/qtbase/lib -lQt5
Bootstrap -LC:\Program Files\Microsoft DirectX SDK (June 2010)\Lib\x86 -luser32
-lole32 -ladvapi32 -lz
g++: error: Files\Microsoft: No such file or directory
g++: error: DirectX: No such file or directory
g++: error: SDK: No such file or directory
g++: error: (June: No such file or directory
g++: error: 2010)\Lib\x86: No such file or directory
Makefile.Release:87: recipe for target '..\..\..\bin\moc.exe' failed
mingw32-make[4]: *** [..\..\..\bin\moc.exe] Error 1
mingw32-make[4]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtbase/src/tools/moc'
Makefile:34: recipe for target 'release' failed
mingw32-make[3]: *** [release] Error 2
mingw32-make[3]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtbase/src/tools/moc'
Makefile:82: recipe for target 'sub-moc-make_first' failed
mingw32-make[2]: *** [sub-moc-make_first] Error 2
mingw32-make[2]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtbase/src'
Makefile:40: recipe for target 'sub-src-make_first' failed
mingw32-make[1]: *** [sub-src-make_first] Error 2
mingw32-make[1]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtbase'
makefile:56: recipe for target 'module-qtbase-make_first' failed
mingw32-make: *** [module-qtbase-make_first] Error 2
</pre></div>

Install the DirectX SDK to a directory with no spaces in the path name, such as C:\DirectX_SDK

If you get errors starting with something like:

<div class="preformatted_console"><pre>
d3d11shader.h:180:30: error: '__out' has not been declared
</pre></div>

Then stop trying to build code that uses DirectX with a none Microsoft toolchain!

What about:

<div class="preformatted_console"><pre>
mingw32-make[4]: Entering directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1
.0/qtjsbackend/src/v8'
python C:/qtbuild/qt-everywhere-opensource-src-5.1.0/qtjsbackend/src/v8/../3rdpa
rty/v8/tools/js2c.py generated-release/libraries.cpp CORE off C:/qtbuild/qt-ever
ywhere-opensource-src-5.1.0/qtjsbackend/src/v8/../3rdparty/v8/src/macros.py ..\3
rdparty\v8\src\runtime.js ..\3rdparty\v8\src\v8natives.js ..\3rdparty\v8\src\arr
ay.js ..\3rdparty\v8\src\string.js ..\3rdparty\v8\src\uri.js ..\3rdparty\v8\src\
math.js ..\3rdparty\v8\src\messages.js ..\3rdparty\v8\src\apinatives.js ..\3rdpa
rty\v8\src\date.js ..\3rdparty\v8\src\regexp.js ..\3rdparty\v8\src\json.js ..\3r
dparty\v8\src\liveedit-debugger.js ..\3rdparty\v8\src\mirror-debugger.js ..\3rdp
arty\v8\src\debug-debugger.js
process_begin: CreateProcess(NULL, python C:/qtbuild/qt-everywhere-opensource-sr
c-5.1.0/qtjsbackend/src/v8/../3rdparty/v8/tools/js2c.py generated-release/librar
ies.cpp CORE off C:/qtbuild/qt-everywhere-opensource-src-5.1.0/qtjsbackend/src/v
8/../3rdparty/v8/src/macros.py ..\3rdparty\v8\src\runtime.js ..\3rdparty\v8\src\
v8natives.js ..\3rdparty\v8\src\array.js ..\3rdparty\v8\src\string.js ..\3rdpart
y\v8\src\uri.js ..\3rdparty\v8\src\math.js ..\3rdparty\v8\src\messages.js ..\3rd
party\v8\src\apinatives.js ..\3rdparty\v8\src\date.js ..\3rdparty\v8\src\regexp.
js ..\3rdparty\v8\src\json.js ..\3rdparty\v8\src\liveedit-debugger.js ..\3rdpart
y\v8\src\mirror-debugger.js ..\3rdparty\v8\src\debug-debugger.js, ...) failed.
make (e=2): The system cannot find the file specified.
Makefile.Release:401: recipe for target 'generated-release/libraries.cpp' failed

mingw32-make[4]: *** [generated-release/libraries.cpp] Error 2
mingw32-make[4]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtjsbackend/src/v8'
Makefile:34: recipe for target 'release' failed
mingw32-make[3]: *** [release] Error 2
mingw32-make[3]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtjsbackend/src/v8'
Makefile:82: recipe for target 'sub-v8-make_first-ordered' failed
mingw32-make[2]: *** [sub-v8-make_first-ordered] Error 2
mingw32-make[2]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtjsbackend/src'
Makefile:39: recipe for target 'sub-src-make_first' failed
mingw32-make[1]: *** [sub-src-make_first] Error 2
mingw32-make[1]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtjsbackend'
makefile:156: recipe for target 'module-qtjsbackend-make_first' failed
mingw32-make: *** [module-qtjsbackend-make_first] Error 2
</pre></div>

Install Python, ensure it is in the path (Too obvious?).

I got an error during make install saying:

<div class="preformatted_console"><pre>
make: *** No rule to make target 'install'. Stop.
</pre></div>

In the qtwebkit-examples directory. Simply added an empty install target in qtwebkit-examples\Makefile.


When building Qt again a second time (With the extra optimisations), I got this:

<div class="preformatted_console"><pre>
mingw32-make[3]: Entering directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1
.0/qtwebkit/Source/WTF'
g++ -c -Wall -Wextra -Wreturn-type -fno-strict-aliasing -Wchar-subscripts -Wform
at-security -Wreturn-type -Wno-unused-parameter -Wno-sign-compare -Wno-switch -W
no-switch-enum -Wundef -Wmissing-noreturn -Winit-self -pipe -fno-keep-inline-dll
export -fdata-sections -ffunction-sections -O3 -fno-exceptions -frtti -DUNICODE
-DBUILDING_QT__=1 -DNDEBUG -DENABLE_3D_RENDERING=1 -DENABLE_BLOB=1 -DENABLE_CHAN
NEL_MESSAGING=1 -DENABLE_CSS_BOX_DECORATION_BREAK=1 -DENABLE_CSS_COMPOSITING=1 -
DENABLE_CSS_EXCLUSIONS=1 -DENABLE_CSS_FILTERS=1 -DENABLE_CSS_IMAGE_SET=1 -DENABL
E_CSS_REGIONS=1 -DENABLE_CSS_STICKY_POSITION=1 -DENABLE_DATALIST_ELEMENT=1 -DENA
BLE_DETAILS_ELEMENT=1 -DENABLE_FAST_MOBILE_SCROLLING=1 -DENABLE_FILTERS=1 -DENAB
LE_FTPDIR=1 -DENABLE_FULLSCREEN_API=1 -DENABLE_GESTURE_EVENTS=1 -DENABLE_ICONDAT
ABASE=1 -DENABLE_IFRAME_SEAMLESS=1 -DENABLE_INPUT_TYPE_COLOR=1 -DENABLE_INSPECTO
R=1 -DENABLE_INSPECTOR_SERVER=1 -DENABLE_JAVASCRIPT_DEBUGGER=1 -DENABLE_LEGACY_N
OTIFICATIONS=1 -DENABLE_LEGACY_VIEWPORT_ADAPTION=1 -DENABLE_LEGACY_VENDOR_PREFIX
ES=1 -DENABLE_LINK_PREFETCH=1 -DENABLE_METER_ELEMENT=1 -DENABLE_MHTML=1 -DENABLE
_MUTATION_OBSERVERS=1 -DENABLE_NOTIFICATIONS=1 -DENABLE_PAGE_VISIBILITY_API=1 -D
ENABLE_PROGRESS_ELEMENT=1 -DENABLE_RESOLUTION_MEDIA_QUERY=1 -DENABLE_REQUEST_ANI
MATION_FRAME=1 -DENABLE_SHARED_WORKERS=1 -DENABLE_SMOOTH_SCROLLING=1 -DENABLE_SQ
L_DATABASE=1 -DENABLE_SVG=1 -DENABLE_SVG_FONTS=1 -DENABLE_TOUCH_ADJUSTMENT=1 -DE
NABLE_TOUCH_EVENTS=1 -DENABLE_WEB_SOCKETS=1 -DENABLE_WEB_TIMING=1 -DENABLE_WORKE
RS=1 -DENABLE_XHR_TIMEOUT=1 -DWTF_USE_TILED_BACKING_STORE=1 -DHAVE_QTQUICK=1 -DH
AVE_QTPRINTSUPPORT=1 -DHAVE_QSTYLE=1 -DHAVE_QTTESTLIB=1 -DWTF_USE_ZLIB=1 -DENABL
E_NETSCAPE_PLUGIN_API=1 -DPLUGIN_ARCHITECTURE_UNSUPPORTED=1 -DWTF_USE_3D_GRAPHIC
S=1 -DENABLE_WEBGL=1 -DENABLE_CSS_SHADERS=1 -DENABLE_ORIENTATION_EVENTS=1 -DENAB
LE_DEVICE_ORIENTATION=1 -DENABLE_VIDEO=1 -DWTF_USE_QT_MULTIMEDIA=1 -DENABLE_TOUC
H_SLIDER=1 -DENABLE_ACCELERATED_2D_CANVAS=0 -DENABLE_ANIMATION_API=0 -DENABLE_BA
TTERY_STATUS=0 -DENABLE_CSP_NEXT=0 -DENABLE_CSS_GRID_LAYOUT=0 -DENABLE_CSS_HIERA
RCHIES=0 -DENABLE_CSS_IMAGE_ORIENTATION=0 -DENABLE_CSS_IMAGE_RESOLUTION=0 -DENAB
LE_CSS_VARIABLES=0 -DENABLE_CSS3_BACKGROUND=0 -DENABLE_CSS3_CONDITIONAL_RULES=0
-DENABLE_CSS3_TEXT=0 -DENABLE_DASHBOARD_SUPPORT=0 -DENABLE_DATAGRID=0 -DENABLE_D
ATA_TRANSFER_ITEMS=0 -DENABLE_DIRECTORY_UPLOAD=0 -DENABLE_DOWNLOAD_ATTRIBUTE=0 -
DENABLE_FILE_SYSTEM=0 -DENABLE_GAMEPAD=0 -DENABLE_GEOLOCATION=0 -DENABLE_HIGH_DP
I_CANVAS=0 -DENABLE_INDEXED_DATABASE=0 -DENABLE_INPUT_SPEECH=0 -DENABLE_INPUT_TY
PE_DATE=0 -DENABLE_INPUT_TYPE_DATETIME=0 -DENABLE_INPUT_TYPE_DATETIMELOCAL=0 -DE
NABLE_INPUT_TYPE_MONTH=0 -DENABLE_INPUT_TYPE_TIME=0 -DENABLE_INPUT_TYPE_WEEK=0 -
DENABLE_LEGACY_CSS_VENDOR_PREFIXES=0 -DENABLE_LINK_PRERENDER=0 -DENABLE_MATHML=0
 -DENABLE_MEDIA_SOURCE=0 -DENABLE_MEDIA_STATISTICS=0 -DENABLE_MEDIA_STREAM=0 -DE
NABLE_MICRODATA=0 -DENABLE_NAVIGATOR_CONTENT_UTILS=0 -DENABLE_NETWORK_INFO=0 -DE
NABLE_PROXIMITY_EVENTS=0 -DENABLE_QUOTA=0 -DENABLE_SCRIPTED_SPEECH=0 -DENABLE_SH
ADOW_DOM=0 -DENABLE_STYLE_SCOPED=0 -DENABLE_SVG_DOM_OBJC_BINDINGS=0 -DENABLE_TEX
T_AUTOSIZING=0 -DENABLE_TEXT_NOTIFICATIONS_ONLY=0 -DENABLE_TOUCH_ICON_LOADING=0
-DENABLE_VIBRATION=0 -DENABLE_VIDEO_TRACK=0 -DENABLE_WEB_AUDIO=0 -DENABLE_XSLT=0
 -DBUILDING_WTF -DBUILDING_WEBKIT -DQT_ASCII_CAST_WARNINGS -DQT_NO_EXCEPTIONS -D
QT_NO_DEBUG -DQT_CORE_LIB -I. -I"C:\qtbuild\openssl-1.0.1j\dist\include" -I"." -
I"wtf" -I"..\..\Source" -I"C:\qtbuild\qt-everywhere-opensource-src-5.1.0\qtwebki
t\Source\include" -I"C:\qtbuild\qt-everywhere-opensource-src-5.1.0\qtscript\incl
ude" -I"C:\qtbuild\qt-everywhere-opensource-src-5.1.0\qtscript\include\QtScript"
 -I"C:\qtbuild\qt-everywhere-opensource-src-5.1.0\qtbase\include" -I"C:\qtbuild\
qt-everywhere-opensource-src-5.1.0\qtbase\include\QtCore" -I".moc\release_static
" -I"C:\qtbuild\qt-everywhere-opensource-src-5.1.0\qtbase\mkspecs\win32-g++" -o
.obj\release_static\DateMath.o wtf\DateMath.cpp
In file included from ./wtf/OwnArrayPtr.h:26:0,
                 from wtf\DateMath.h:52,
                 from wtf\DateMath.cpp:73:
./wtf/NullPtr.h:52:1: warning: identifier 'nullptr' is a keyword in C++11 [-Wc++
0x-compat]
 extern WTF_EXPORT_PRIVATE std::nullptr_t nullptr;
 ^
In file included from ./wtf/unicode/Unicode.h:34:0,
                 from ./wtf/text/ASCIIFastPath.h:30,
                 from ./wtf/text/WTFString.h:28,
                 from wtf\DateMath.h:54,
                 from wtf\DateMath.cpp:73:
./wtf/unicode/icu/UnicodeIcu.h:27:27: fatal error: unicode/uchar.h: No such file
 or directory
 #include <unicode/uchar.h>
                           ^
compilation terminated.
Makefile.WTF.Release:1238: recipe for target '.obj/release_static/DateMath.o' fa
iled
mingw32-make[3]: *** [.obj/release_static/DateMath.o] Error 1
mingw32-make[3]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtwebkit/Source/WTF'
Makefile.WTF:34: recipe for target 'release' failed
mingw32-make[2]: *** [release] Error 2
mingw32-make[2]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtwebkit/Source/WTF'
</pre></div>

In which case, copy all the headers from qtwebkit\Source\WTF\icu\unicode\ to qtwebkit\Source\WTF\wtf\unicode\, but don't replace any files that have the same name because they are different. Not a very clean way to go about things, but the headers that are included are then found, so it gets the build going again.

And this one:

<div class="preformatted_console"><pre>
mingw32-make[4]: Entering directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1
.0/qtwebkit/Source/JavaScriptCore'
ruby C:/qtbuild/qt-everywhere-opensource-src-5.1.0/qtwebkit/Source/JavaScriptCor
e/offlineasm/generate_offset_extractor.rb llint\LowLevelInterpreter.asm LLIntDes
iredOffsets.h
process_begin: CreateProcess(NULL, ruby C:/qtbuild/qt-everywhere-opensource-src-
5.1.0/qtwebkit/Source/JavaScriptCore/offlineasm/generate_offset_extractor.rb lli
nt\LowLevelInterpreter.asm LLIntDesiredOffsets.h, ...) failed.
make (e=2): The system cannot find the file specified.
Makefile.LLIntOffsetsExtractor.Release:146: recipe for target 'LLIntDesiredOffse
ts.h' failed
mingw32-make[4]: *** [LLIntDesiredOffsets.h] Error 2
mingw32-make[4]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtwebkit/Source/JavaScriptCore'
Makefile.LLIntOffsetsExtractor:38: recipe for target 'release-all' failed
mingw32-make[3]: *** [release-all] Error 2
mingw32-make[3]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtwebkit/Source/JavaScriptCore'
Makefile.JavaScriptCore:38: recipe for target 'sub-LLIntOffsetsExtractor-pro-mak
e_first-ordered' failed
mingw32-make[2]: *** [sub-LLIntOffsetsExtractor-pro-make_first-ordered] Error 2
mingw32-make[2]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtwebkit/Source/JavaScriptCore'
Makefile:88: recipe for target 'sub-Source-JavaScriptCore-JavaScriptCore-pro-mak
e_first-ordered' failed
mingw32-make[1]: *** [sub-Source-JavaScriptCore-JavaScriptCore-pro-make_first-or
dered] Error 2
mingw32-make[1]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtwebkit'
makefile:310: recipe for target 'module-qtwebkit-make_first' failed
mingw32-make: *** [module-qtwebkit-make_first] Error 2
</pre></div>

If you read the error, yes, install Ruby is the answer.

Again with the unicode headers in webkit:

<div class="preformatted_console"><pre>
g++ -c -Wall -Wextra -Wreturn-type -fno-strict-aliasing -Wchar-subscripts -Wform
at-security -Wreturn-type -Wno-unused-parameter -Wno-sign-compare -Wno-switch -W
no-switch-enum -Wundef -Wmissing-noreturn -Winit-self -pipe -fno-keep-inline-dll
export -O2 -fdata-sections -ffunction-sections -fno-exceptions -frtti -DUNICODE
-DBUILDING_QT__=1 -DNDEBUG -DENABLE_3D_RENDERING=1 -DENABLE_BLOB=1 -DENABLE_CHAN
NEL_MESSAGING=1 -DENABLE_CSS_BOX_DECORATION_BREAK=1 -DENABLE_CSS_COMPOSITING=1 -
DENABLE_CSS_EXCLUSIONS=1 -DENABLE_CSS_FILTERS=1 -DENABLE_CSS_IMAGE_SET=1 -DENABL
E_CSS_REGIONS=1 -DENABLE_CSS_STICKY_POSITION=1 -DENABLE_DATALIST_ELEMENT=1 -DENA
BLE_DETAILS_ELEMENT=1 -DENABLE_FAST_MOBILE_SCROLLING=1 -DENABLE_FILTERS=1 -DENAB
LE_FTPDIR=1 -DENABLE_FULLSCREEN_API=1 -DENABLE_GESTURE_EVENTS=1 -DENABLE_ICONDAT
ABASE=1 -DENABLE_IFRAME_SEAMLESS=1 -DENABLE_INPUT_TYPE_COLOR=1 -DENABLE_INSPECTO
R=1 -DENABLE_INSPECTOR_SERVER=1 -DENABLE_JAVASCRIPT_DEBUGGER=1 -DENABLE_LEGACY_N
OTIFICATIONS=1 -DENABLE_LEGACY_VIEWPORT_ADAPTION=1 -DENABLE_LEGACY_VENDOR_PREFIX
ES=1 -DENABLE_LINK_PREFETCH=1 -DENABLE_METER_ELEMENT=1 -DENABLE_MHTML=1 -DENABLE
_MUTATION_OBSERVERS=1 -DENABLE_NOTIFICATIONS=1 -DENABLE_PAGE_VISIBILITY_API=1 -D
ENABLE_PROGRESS_ELEMENT=1 -DENABLE_RESOLUTION_MEDIA_QUERY=1 -DENABLE_REQUEST_ANI
MATION_FRAME=1 -DENABLE_SHARED_WORKERS=1 -DENABLE_SMOOTH_SCROLLING=1 -DENABLE_SQ
L_DATABASE=1 -DENABLE_SVG=1 -DENABLE_SVG_FONTS=1 -DENABLE_TOUCH_ADJUSTMENT=1 -DE
NABLE_TOUCH_EVENTS=1 -DENABLE_WEB_SOCKETS=1 -DENABLE_WEB_TIMING=1 -DENABLE_WORKE
RS=1 -DENABLE_XHR_TIMEOUT=1 -DWTF_USE_TILED_BACKING_STORE=1 -DHAVE_QTQUICK=1 -DH
AVE_QTPRINTSUPPORT=1 -DHAVE_QSTYLE=1 -DHAVE_QTTESTLIB=1 -DWTF_USE_ZLIB=1 -DENABL
E_NETSCAPE_PLUGIN_API=1 -DPLUGIN_ARCHITECTURE_UNSUPPORTED=1 -DWTF_USE_3D_GRAPHIC
S=1 -DENABLE_WEBGL=1 -DENABLE_CSS_SHADERS=1 -DENABLE_ORIENTATION_EVENTS=1 -DENAB
LE_DEVICE_ORIENTATION=1 -DENABLE_VIDEO=1 -DWTF_USE_QT_MULTIMEDIA=1 -DENABLE_TOUC
H_SLIDER=1 -DENABLE_ACCELERATED_2D_CANVAS=0 -DENABLE_ANIMATION_API=0 -DENABLE_BA
TTERY_STATUS=0 -DENABLE_CSP_NEXT=0 -DENABLE_CSS_GRID_LAYOUT=0 -DENABLE_CSS_HIERA
RCHIES=0 -DENABLE_CSS_IMAGE_ORIENTATION=0 -DENABLE_CSS_IMAGE_RESOLUTION=0 -DENAB
LE_CSS_VARIABLES=0 -DENABLE_CSS3_BACKGROUND=0 -DENABLE_CSS3_CONDITIONAL_RULES=0
-DENABLE_CSS3_TEXT=0 -DENABLE_DASHBOARD_SUPPORT=0 -DENABLE_DATAGRID=0 -DENABLE_D
ATA_TRANSFER_ITEMS=0 -DENABLE_DIRECTORY_UPLOAD=0 -DENABLE_DOWNLOAD_ATTRIBUTE=0 -
DENABLE_FILE_SYSTEM=0 -DENABLE_GAMEPAD=0 -DENABLE_GEOLOCATION=0 -DENABLE_HIGH_DP
I_CANVAS=0 -DENABLE_INDEXED_DATABASE=0 -DENABLE_INPUT_SPEECH=0 -DENABLE_INPUT_TY
PE_DATE=0 -DENABLE_INPUT_TYPE_DATETIME=0 -DENABLE_INPUT_TYPE_DATETIMELOCAL=0 -DE
NABLE_INPUT_TYPE_MONTH=0 -DENABLE_INPUT_TYPE_TIME=0 -DENABLE_INPUT_TYPE_WEEK=0 -
DENABLE_LEGACY_CSS_VENDOR_PREFIXES=0 -DENABLE_LINK_PRERENDER=0 -DENABLE_MATHML=0
 -DENABLE_MEDIA_SOURCE=0 -DENABLE_MEDIA_STATISTICS=0 -DENABLE_MEDIA_STREAM=0 -DE
NABLE_MICRODATA=0 -DENABLE_NAVIGATOR_CONTENT_UTILS=0 -DENABLE_NETWORK_INFO=0 -DE
NABLE_PROXIMITY_EVENTS=0 -DENABLE_QUOTA=0 -DENABLE_SCRIPTED_SPEECH=0 -DENABLE_SH
ADOW_DOM=0 -DENABLE_STYLE_SCOPED=0 -DENABLE_SVG_DOM_OBJC_BINDINGS=0 -DENABLE_TEX
T_AUTOSIZING=0 -DENABLE_TEXT_NOTIFICATIONS_ONLY=0 -DENABLE_TOUCH_ICON_LOADING=0
-DENABLE_VIBRATION=0 -DENABLE_VIDEO_TRACK=0 -DENABLE_WEB_AUDIO=0 -DENABLE_XSLT=0
 -DQT_NO_EXCEPTIONS -I. -I"C:\qtbuild\openssl-1.0.1j\dist\include" -I"C:\qtbuild
\qt-everywhere-opensource-src-5.1.0\qtbase\include" -I"C:\qtbuild\qt-everywhere-
opensource-src-5.1.0\qtbase\include\QtCore" -I"." -I"..\..\Source" -I"..\WTF" -I
"assembler" -I"bytecode" -I"bytecompiler" -I"heap" -I"dfg" -I"debugger" -I"disas
sembler" -I"interpreter" -I"jit" -I"llint" -I"parser" -I"profiler" -I"runtime" -
I"tools" -I"yarr" -I"API" -I"ForwardingHeaders" -I"C:\qtbuild\qt-everywhere-open
source-src-5.1.0\qtwebkit\Source\JavaScriptCore\generated" -I"..\WTF" -I"..\..\S
ource" -I"C:\qtbuild\qt-everywhere-opensource-src-5.1.0\qtwebkit\Source\include"
 -I"C:\qtbuild\qt-everywhere-opensource-src-5.1.0\qtscript\include" -I"C:\qtbuil
d\qt-everywhere-opensource-src-5.1.0\qtscript\include\QtScript" -I"C:\qtbuild\qt
-everywhere-opensource-src-5.1.0\qtbase\mkspecs\win32-g++" -o .obj\release_stati
c\LLIntOffsetsExtractor.o llint\LLIntOffsetsExtractor.cpp
In file included from ..\WTF/wtf/PassRefPtr.h:26:0,
                 from ..\WTF/wtf/RefPtr.h:28,
                 from ..\WTF/wtf/HashFunctions.h:24,
                 from ..\WTF/wtf/HashTraits.h:24,
                 from ..\WTF/wtf/HashTable.h:29,
                 from ..\WTF/wtf/HashMap.h:24,
                 from runtime/JSValue.h:31,
                 from heap/HandleTypes.h:29,
                 from runtime/WriteBarrier.h:30,
                 from runtime/PropertyStorage.h:29,
                 from runtime/IndexingHeader.h:29,
                 from runtime/ArrayConventions.h:24,
                 from runtime/JSArray.h:24,
                 from bytecode/ArrayProfile.h:29,
                 from llint\LLIntOffsetsExtractor.cpp:28:
..\WTF/wtf/NullPtr.h:52:1: warning: identifier 'nullptr' is a keyword in C++11 [
-Wc++0x-compat]
 extern WTF_EXPORT_PRIVATE std::nullptr_t nullptr;
 ^
In file included from ..\WTF/wtf/unicode/Unicode.h:34:0,
                 from ..\WTF/wtf/StringHasher.h:24,
                 from ..\WTF/wtf/text/StringImpl.h:30,
                 from ..\WTF/wtf/text/AtomicStringImpl.h:24,
                 from ..\WTF/wtf/text/AtomicString.h:24,
                 from ..\WTF/wtf/text/StringHash.h:25,
                 from heap/SlotVisitor.h:32,
                 from heap/Heap.h:37,
                 from runtime/WriteBarrier.h:31,
                 from runtime/PropertyStorage.h:29,
                 from runtime/IndexingHeader.h:29,
                 from runtime/ArrayConventions.h:24,
                 from runtime/JSArray.h:24,
                 from bytecode/ArrayProfile.h:29,
                 from llint\LLIntOffsetsExtractor.cpp:28:
..\WTF/wtf/unicode/icu/UnicodeIcu.h:27:27: fatal error: unicode/uchar.h: No such
 file or directory
 #include <unicode/uchar.h>
                           ^
compilation terminated.
Makefile.LLIntOffsetsExtractor.Release:592: recipe for target '.obj/release_stat
ic/LLIntOffsetsExtractor.o' failed
mingw32-make[4]: *** [.obj/release_static/LLIntOffsetsExtractor.o] Error 1
mingw32-make[4]: Leaving directory 'C:/qtbuild/qt-everywhere-opensource-src-5.1.
0/qtwebkit/Source/JavaScriptCore'
</pre></div>

At this point I wondered why the build had worked the first time through, but kept falling over the second time. When checking, I found that in the first build, Webkit had been disabled because ICU was disabled. After cleaning the source down and starting again, this time the configure script had decided it would try and build Webkit anyway.

Upon thinking about this, it was rather obvious that fixing header file inclusion issues would only ever lead to linker errors due to purposefully absent ICU libraries. Thats when I added the `-skip qtwebkit` option.

