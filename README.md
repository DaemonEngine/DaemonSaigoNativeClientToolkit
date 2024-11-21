# Dæmon Saigo Native Client Toolkit

Saigo is a new toolchain for compiling Native Client applications, Native Client was publicly abandonned by Google in 2020 in favor of WebAssembly, but Google silently continued development in the form of a new toolchain named Saigo. This project is to make possible to rebuild Native Client compilers and loaders for usage with the [Dæmon game engine](https://github.com/DaemonEngine/Daemon).


## How-to

```
mkdir build && cd build
cmake ..
make -j32
```

A compiler toolchain can then be found in `build/prefix`.

The binutils build relies on autotools and GNU Make, so `gmake` has to be used instead of `make` on BSD systems.

Replace 32 with the amount of cores your computer provides. Rebuilding Clang requires a powerful computer (or very slow compilation is expected).

## History

But despite the public depredaction and abandonment, Google continued development in the form of a brand new toolchain named Saigo.

History of NaCl toolchains, from older to newer:

- PNaCl GCC (GCC 4.4.3)
- PNaCl Clang (LLVM 3.6)
- Saigo Clang (LLVM 19 at the time of writing)

The Saigo toolchain is based on Clang and is frequently rebased over current LLVM, bringing latest Clang and latest C++ standards to NaCl, it also requires a special branch of binutils.

The libc++ is the one from the same recent LLVM Saigo is based on, and should be easy to build, unfortunately building some parts of the libc still requires the old GCC, which is very old and now hard to build, and Google themselves don't rebuild it in their scripts.


## Purpose

Purpose of this project is to provide a way to build native binaries for the toolchain running on developer's computer without relying on Google-provided binaries, this includes clang and the binutils. This would also allow to run the toolchain on systems not supported by Google. Google only build Saigo for Linux x86, and Google only built PNaCl-clang for linux-amd64, windows-amd64, windows-i686 and macos-am64. 

For the nexe libc and libc++, this project aims to repackage the Google pre-compiled libraries in order to reach so minimimum viable product state. This precompiled code is meant to be executed in the NativeClient sandbox and then doesn't require the same level of trust.

Contributions making possible to fully rebuild the libc without any Google-provided binaries is welcome, though, but not as prioritary as providing a fully-rebuildable NaCl compiler toolchain environment.

Rebuilding the NaCl loader too is planned.


## Supported nexe target platforms

Unlike PNaCl, Saigo doesn't support MIPS, here are the supported targets:

- i686
- amd64
- arm

Unlike PNaCl, Saigo doesn't compile to pexe, so the code should be rebuilt for every target platform instead of compiling one pexe once and translating it to multiple nexe  after that. The pexe to nexe translation being very slow, the iterative rebuild of three targets is faster anyway.

It also means precompiled pexe static libraries for various common libraries provided by Google (FreeType, etc.) cannot be used. Migrating a project to Saigo may then require to rebuild some dependencies.


## Limitations

Unlike PNaCl, Saigo doesn't support exceptions, the Google scripts to build the libs actually do build some stuff for exceptions, but it doesn't work. Support for `setjmp`/`longjmp` is working.
