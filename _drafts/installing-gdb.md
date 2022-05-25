---
title: "Tools we use: installing `gdb` for ARM"
description: Demonstrate a few methods of installing gdb for ARM
author: noah
image: img/<post-slug>/cover.png # 1200x630
---

<!-- excerpt start -->

In this mini article, I'll be going on a few strategies and nuances around
getting a copy of GDB installed for debugging ARM chips.

<!-- excerpt end -->

{% include newsletter.html %}

{% include toc.html %}

## GDB for ARM

GDB, the GNU Project Debugger, is by default typically compiled to target the
same architecture as the host system. When we talk about "host" and "target
architectures, what we mean is:

1. the **host** architecture: where the GDB program itself is run
2. the **target** architecture: where the program being debugged is run

Debugging a program that's running on the same machine as the host might look
like this:

[![img/installing-gdb/gdb_host_target.drawio.svg](/img/installing-gdb/gdb_host_target.drawio.svg)](/img/installing-gdb/gdb_host_target.drawio.svg)

In embedded engineering, we often want to target a foreign architecture (eg, the
embedded device, connected to some debug probe). That setup might look like
this:

[![img/installing-gdb/gdb_cross_target.drawio.svg](/img/installing-gdb/gdb_cross_target.drawio.svg)](/img/installing-gdb/gdb_cross_target.drawio.svg)

We need a version of GDB that supports the target architecture of the program
being debugged.

## Summary of Strategies

I'm going to go through the various ways to install GDB with ARM support, but
first here's a table summarizing the approaches:

| Strategy               | Binary or source install? | Pinned to version | Python support        |
| ---------------------- | ------------------------- | ----------------- | --------------------- |
| Build from source      | 📁source                  | ✅ yes            | ✅ (configurable)     |
| Binaries from ARM      | 📦binary                  | ✅ yes            | ⚠️ varies per version |
| System package manager | 📦binary                  | ⚠️ varies         | ⚠️ varies             |
| Conda                  | 📦binary                  | ✅ yes            | ✅ yes                |
| Docker                 | 📦binary                  | ✅ yes            | ✅ yes                |
| SDK-specific tools     | 📦binary                  | ⚠️ varies         | ✅ (usually yes)      |

## Build from source

[Building a custom copy of GDB]({% post_url 2020-04-08-gnu-binutils %}) is one
way to enable support for the necessary architecture.

You might for example build GDB enabling all targets:

```bash
❯ ./configure
  --target="arm-elf-linux"
  --enable-targets=all
  --with-python
❯ make -j $(nproc)
```

This definitely results in a nicely tuned copy of GDB- you can select only the
features needed, enable different optimizations, select the correct Python
version to integrate, etc., and are in full control of your software supply
chain!

The downside for from-source builds (in general):

- can be challenging to build for Windows or other non-Unix platforms
- need to maintain build tooling + binary repositories
- using custom builds, you might encounter edge cases or warts that have been
  solved in more widely used copies of the same software

> Quick note- if you want to see which architectures a given `gdb` binary
> supports, here's a shell "one-liner" that can be useful (taken from
> [here](https://sourceware.org/git/?p=binutils-gdb.git;a=blob;f=gdb/gdb_buildall.sh;hb=4a94e36819485cdbd50438f800d1e478156a4889#l206)):
>
> ```bash
> gdb --batch -nx --ex 'set architecture' 2>&1 | \
>   tail -n 1 | \
>   sed 's/auto./\n/g' | \
>   sed 's/,/\n/g' | \
>   sed 's/Requires an argument. Valid arguments are/\n/g' | \
>   sed '/^[ ]*$/d' | sort | uniq
> ```

## Binaries from ARM

ARM ships full prebuilt GCC + Binutils toolchains for Linux, Windows, and Mac.
These were originally hosted on launchpad.net, but were moved here around 2016:

<https://developer.arm.com/downloads/-/gnu-rm>

In 2022, the downloads relocated again to this page:

<https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/downloads>

These packages are very handy- they include the following components:

- GCC compiler
- all binutils
  - GDB
  - LD
  - objdump, etc
- prebuilt Newlib C library
- prebuilt "Newlib-Nano" reduced size C library
- various example linker scripts and startup files

Another benefit- you can have developers on Windows/Linux/Mac all using the same
version of the compiler (doesn't necessarily guarantee bit-for-bit reproducible
builds, but chances are pretty good).

The same tools can also easily be used in whatever CI system you are using.

There is one gotcha with these packages, though- the python support is whatever
was configured at build time by ARM. This means if you're using [GDB's python
API]({% post_url 2019-07-02-automate-debugging-with-gdb-python-api %}), you'll
need to make sure your python scripts/dependencies/interpreter matches the
interpreter embedded into GDB.

And in the latest release, python support was dropped in the Mac release:

```bash
# check which dynamic libraries the mac arm-none-eabi-gdb binary depends on

# v11.2-2022.02 (no python support 😔. note the *gdb-py binary is absent from
# the archive)
❯ llvm-otool-14 -L ~/Downloads/gcc-arm-11.2-2022.02-darwin-x86_64-arm-none-eabi/bin/arm-none-eabi-gdb
/home/noah/Downloads/gcc-arm-11.2-2022.02-darwin-x86_64-arm-none-eabi/bin/arm-none-eabi-gdb:
        /usr/lib/libncurses.5.4.dylib (compatibility version 5.4.0, current version 5.4.0)
        /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1281.100.1)
        /usr/lib/libiconv.2.dylib (compatibility version 7.0.0, current version 7.0.0)
        /usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 902.1.0)

# v10.3-2021.10 (python2.7! in 2021!)
❯ llvm-otool-14 -L ~/Downloads/gcc-arm-none-eabi-10.3-2021.10-mac/bin/arm-none-eabi-gdb-py
/home/noah/Downloads/gcc-arm-none-eabi-10.3-2021.10-mac/bin/arm-none-eabi-gdb-py:
        /usr/lib/libncurses.5.4.dylib (compatibility version 5.4.0, current version 5.4.0)
        /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1252.250.1)
        /System/Library/Frameworks/Python.framework/Versions/2.7/Python (compatibility version 2.7.0, current version 2.7.10)
        /System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation (compatibility version 150.0.0, current version 1575.17.0)
        /usr/lib/libiconv.2.dylib (compatibility version 7.0.0, current version 7.0.0)
        /usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 400.9.4)

# still present in the linux version of v11.2-2022.02 though. But it's a
# somewhat venerable python3.6, which I don't have installed right now 😅
❯ ldd ~/Downloads/gcc-arm-11.2-2022.02-x86_64-arm-none-eabi/bin/arm-none-eabi-gdb
        linux-vdso.so.1 (0x00007ffd3fd4d000)
        libncursesw.so.5 => /lib/x86_64-linux-gnu/libncursesw.so.5 (0x00007f9f6c717000)
        libtinfo.so.5 => /lib/x86_64-linux-gnu/libtinfo.so.5 (0x00007f9f6c6e8000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f9f6c6e3000)
        libpython3.6m.so.1.0 => not found
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f9f6c6de000)
        libutil.so.1 => /lib/x86_64-linux-gnu/libutil.so.1 (0x00007f9f6c6d7000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f9f6c5f0000)
        libexpat.so.1 => /lib/x86_64-linux-gnu/libexpat.so.1 (0x00007f9f6c5bf000)
        libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f9f6c393000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f9f6c373000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f9f6c14b000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f9f6c764000)
```

This could potentially be a pretty serious problem if there is some tooling or
scripting that uses the python API and the release for your host system doesn't
have python enabled.

## System package manager

## Conda

## Docker

## SDK-specific tools

<!-- Interrupt Keep START -->

{% include newsletter.html %}

{% include submit-pr.html %}

<!-- Interrupt Keep END -->

{:.no_toc}

## References

<!-- prettier-ignore-start -->
[^reference_key]: [Post Title](https://example.com)
<!-- prettier-ignore-end -->

```

```
