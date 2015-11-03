---
layout: post
title: "(Un)building Software #2"
author: geyslan
modified:
categories: blog
excerpt: ""
tags: [linux, code, assembly, c, elf, reverse engineering, english]
share: true
image:
  feature:
  credit: #name of the person or site you want to credit
  creditlink: #url to their site or licensing
date: 2015-11-02T09:27:20-03:00
---

At the end of the first post, I said that this sequence would be a
practical approach of reverse engineering which would allow an
objective understanding of ELF structure. However, I chose to invert
this order, and now we'll begin directly by ELF, whereas the analysis
of its structure, even minimally, is a reversing by itself.

<!--more-->

I need to mention the fixes that I made in the
[first](/blog/unbuilding-software-english-version/) post. Thanks to
[Ygor Parreira](http://www.linkedin.com/pub/ygor-da-rocha-parreira/12/248/b75)
(thanks, dmr!), I have identified outdated information on the process
compilation description.  Then to not spoil your study, the re-reading
is mandatory.

The files of this post are available on
[GitHub](https://github.com/geyslan/hb/tree/master/desconstruindo).

##ELF (Pointed Ears)

The ELF (Executable and Linking Format) is a format (structure) that
specifies the composition and organization of an object file
(resulting binary representation of the assembly process) for the
latter to be functional to the operating system that uses it. It's, in
short, a map that allows the proper creation and usage, through the
linker and loader, of the files with that pattern.

Originally developed and published by USL (UNIX System Laboratories) as
part of ABI, the ELF became default in various Unix-like operating systems,
replacing older formats such as a.out and COFF. Nowadays it's also used
on non-Unix operating systems such as OpenVMS, BeOS and Haiku, as well as in
video games, mobile phones, tablets, routers, televisions etc.

As regards the
[System V - Application Binary Interface (ABI), Ed. 4.1](http://www.sco.com/developers/devspecs/gabi41.pdf)
introduction (p. 45) (document that describes the interface between
the program and the operating system or other program), I do one
caveat to the statement

> "Created by the assembler 'and' link editor"

whereas an opposition to this rule has been demonstrated in the
construction of the [crackme.03](/blog/crackme-03-source-code), in
which the linking was neglected. Anyway, in a normal build process
it's correct to say that the linking phase will be present.

## ELF Kinds (Middle-Earth, D&D, ...)

Let's talk a little about the most common types of ELF object files.

{% include imagehb.html url="/images/blog/unbuilding-software-2/ELF.Types.png" caption="Object File Types" %}

### Relocatable

The relocatable object file has code and data ready for combination
with other object files that shall compose an executable or a shared
object.

See the programa1.o.

{% highlight console %}
$ gcc programa1.c -m32 -c

$ file programa1.o
programa1.o: ELF 32-bit LSB  relocatable, Intel 80386, version 1 (SYSV), not stripped

$ readelf-h programa1.o | grep Type
 Type:                              REL (Relocatable file)

$ readelf -r programa1.o
Relocation section '.rel.text' at offset 0x3ec contains 2 entries:
Offset     Info    Type            Sym.Value  Sym. Name
0000000c  00000501 R_386_32          00000000   .rodata
00000011  00000a02 R_386_PC32        00000000   puts

Relocation section '.rel.eh_frame' at offset 0x3fc contains 1 entries:
Offset     Info    Type            Sym.Value  Sym. Name
00000020  00000202 R_386_PC32        00000000   .text
{% endhighlight %}

In this first example, the gcc compiled and set up the relocatable binary programa1.o
which still hasn't the addresses for execution defined in its structure (ELF)
as well still hasn't its references symbols resolved.

Using readelf we have sure that the addresses were not defined.

{% highlight console %}
$ readelf -S programa1.o
There are 13 section headers, starting at offset 0x11c:

Section Headers:
 [Nr] Name          Type            Addr     Off    Size   ES Flg Lk Inf Al
 [ 0]               NULL            00000000 000000 000000 00      0   0  0
 [ 1] .text         PROGBITS        00000000 000034 00001c 00  AX  0   0  4
 [ 2] .rel.text     REL             00000000 0003ec 000010 08     11   1  4
 [ 3] .data         PROGBITS        00000000 000050 000000 00  WA  0   0  4
 [ 4] .bss          NOBITS          00000000 000050 000000 00  WA  0   0  4
 [ 5] .rodata       PROGBITS        00000000 000050 00000c 00   A  0   0  1
...
{% endhighlight %}

The [nm](http://linux.die.net/man/1/nm) give us the list with all
symbols to be solved yet.

{% highlight console %}
$ nm -a programa1.o
00000000 b .bss
...
00000000 d .data
...
00000000 t .text
00000000 T main
...
{% endhighlight %}

This is other example using the relocatable programa2.o (programa1
assembly version)

{% highlight console %}
$ nasm -f elf32 programa2.asm

$ readelf -r programa2.o
Relocation section '.rel.text' at offset 0x260 contains 1 entries:
Offset     Info    Type            Sym.Value  Sym. Name
0000000b  00000201 R_386_32          00000000   .data
{% endhighlight %}

### Executable

This object file type contains the information needed to create a
respective process image via the function (syscall) exec.

Let's go ahead introducing the
[programa1](https://github.com/geyslan/hb/blob/master/desconstruindo/programa1.c).

{% highlight console %}
$ gcc programa1.c -m32 -o programa1
$ ./programa1
Hello World

$ file programa1
programa1: ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=df0b281079aace57fec72570a37ba345c7679a59, not stripped

$ readelf -S programa1
There are 30 section headers, starting at offset 0x7f0:

Section Headers:

 [Nr] Name           Type           Addr     Off    Size   ES Flg Lk Inf Al
...
 [12] .plt           PROGBITS       080482c0 0002c0 000040 04  AX  0   0 16
 [13] .text          PROGBITS       08048300 000300 000194 00  AX  0   0 16
 [14] .fini          PROGBITS       08048494 000494 000014 00  AX  0   0  4
 [15] .rodata        PROGBITS       080484a8 0004a8 000014 00   A  0   0  4
...
{% endhighlight %}

Above we can see that the object file of type executable has a
relocatable structure.

We can confirm that, also, through the ELF header. One thing to be
looked is that the relocatable object has no entry point.

{% highlight console %}
$ readelf -h programa1.o | grep Entry
  Entry point address:               0x0
{% endhighlight %}

However, it owns this address when it's executable.

{% highlight console %}
$ readelf -h programa1 | grep Entry
  Entry point address:               0x8048300
{% endhighlight %}

In the case of programa1, the linking (collect2) binded (through
symbol relations) the local and dynamic (libraries) references and
relocated the data structures, making the object file ready to
execute.

Further with the tool [ldd](http://linux.die.net/man/1/ldd) we have a
list of the the shared libraries necessary to the executable.

{% highlight console %}
$ ldd programa1
linux-gate.so.1 (0xf76ea000)
libc.so.6 => /usr/lib32/libc.so.6 (0xf751b000)
/lib/ld-linux.so.2 (0xf76eb000)
{% endhighlight %}

We can see that programa1.c uses the libc shared library (by calling
the printf function).

The
[programa2](https://github.com/geyslan/hb/blob/master/desconstruindo/programa2.asm)
(assembly), however, has its pectuliarities.

{% highlight console %}
$ ld -m elf_i386 programa2.o -o programa2
./programa2
Hello World

$ readelf -r programa2
There are no relocations in this file.

$ file programa2
programa2: ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), statically linked, not stripped

$ ldd programa2
not a dynamic executable
{% endhighlight %}

The ldd shows no shared library, since programa2 makes use of syscalls
only. This fact indicates that wasn't necessary to make the
combination with other object files, remaining only other procedures
such as local symbol reference resolution and relocation. Worth
mentioning that the program1, unlike programa2, will also be
[dynamically](https://en.wikipedia.org/wiki/Dynamic_linker) linked in
each new instantiation, because of its undefined symbols.

### Shared Object

This object file holds suitable code and data for two linking situations.

#### Shared object combined with other Shared and/or Relocatable object

At first, let's create the shared object file libfoo1.so.

{% highlight console %}
$ gcc -c -fPIC -m32 foo1.c -o foo1.o
$ readelf -h foo1.o  | grep Type
  Type:                              REL (Relocatable file)

$ gcc -shared -m32 -o libfoo1.so foo1.o
$ readelf -h libfoo1.so | grep Type
 Type:                              DYN (Shared object file)
{% endhighlight %}

Below is the creation from the combination of the shared object file
libfoo1.so with the relocatable foo2.o.

{% highlight console %}
$ gcc -c -fPIC -m32 foo2.c -o foo2.o
$ gcc -shared -m32 -o libfoo2.so foo2.o libfoo1.so
$ readelf -h libfoo2.so | grep Type
  Type:                              DYN (Shared object file)
{% endhighlight %}


And finally combining the two shared objects, libfoo2.so and
libfoo1.so.

{% highlight console %}
$ gcc -shared -m32 -o libfoo3.so libfoo2.so libfoo1.so
$ readelf -h libfoo3.so | grep Type
  Type:                              DYN (Shared object file)
{% endhighlight %}

#### Combined with an Executable or other Shared object for process image creation by the dynamic linker

This is the case of our program1 that when running is combined, prior
to its start, with the other shared object files through
ld-linux.so. This also occurs with shared libraries to be loaded by
the system.

### Core Dump

A core [dump](http://linux.die.net/man/5/core)
([memory](https://en.wikipedia.org/wiki/Core_dump) dump, system dump)
is an object file containing the memory status of a particular
program, produced by the system usually when that ends abnormally
([crashed](https://en.wikipedia.org/wiki/Crash_%28computing%29)).

Let's see it as root.

{% highlight console %}
$ gcc -m32 coredump.c -o coredump

$ ulimit -c
0
$ ulimit -c unlimited
$ ulimit -c
unlimited

$ ./coredump
Segmentation fault (core dumped)
{% endhighlight %}

Generally, the dump is saved in the current folder with the name
core. The location and name of the dump format can be changed in the
/proc/sys/kernel/core_pattern. More information: man core.

{% highlight console %}
$ file core
core: ELF 32-bit LSB  core file Intel 80386, version 1 (SYSV), SVR4-style, from './coredump'
{% endhighlight %}

If you are using Arch Linux, as was my case, it's necessary to extract
the core dump from the journaling system used by systemd.

{% highlight console %}
$ systemd-coredumpctl | tail
...
Sex 2013-05-31 07:41:01 BRT    447     0     0  11 /home/uzumaki/git/hb/desconstruindo/coredump

$ systemd-coredumpctl dump -o core
TIME                           PID   UID   GID SIG EXE
Sex 2013-05-31 07:41:01 BRT    447     0     0  11 /home/uzumaki/git/hb/desconstruindo/coredump
More than one entry matches, ignoring rest.

$ file core
core: ELF 32-bit LSB  core file Intel 80386, version 1 (SYSV), SVR4-style, from './coredump'
{% endhighlight %}

An analysis example of the core dump would be with gdb to investigate the
cause of the crash.

{% highlight console %}
$ gdb -q ./coredump core
Reading symbols from /home/uzumaki/git/hb/desconstruindo/coredump...(no debugging symbols found)...done.
[New LWP 447]

warning: Could not load shared library symbols for linux-gate.so.1.
Do you need "set solib-search-path" or "set sysroot"?
Core was generated by `./coredump'.
Program terminated with signal 11, Segmentation fault.
#0  0x4c4b4a49 in ?? ()
(gdb) backtrace
#0  0x4c4b4a49 in ?? ()
#1  0x504f4e4d in ?? ()
#2  0x54535251 in ?? ()
#3  0x58575655 in ?? ()
#4  0xf7005a59 in ?? ()
...
{% endhighlight %}

As can be seen from the above the segmentation fault occurred by the
characters overflow.

## ELF Structure (Crystal Bones, Wood Bones, ...)

The ELF format provides parallel views of the binary content which
reflect the different needs when linking and running a program. First
we will study the Linking View.

{% include imagehb.html url="/images/blog/unbuilding-software-2/ELF.Format.png" caption="ELF Views" %}

There is only one component with a fixed location in entire structure,
the ELF Header that is the zero offset of the object file. It stores
information that describes the organization of the entire file.

The Program header table tells the system how to create a process
image, so it's a required component in executable files and shared
object; the relocatable didn't make use of it.

The Sections include the bulk information of the object file used in
linking view, such as instructions, data, symbol table, relocation
information, among others.

A section header table contains descriptive information of the
sections of the object. In it, for each section, there is an entry
that provides info such as name, size, among others. In linkage the
object file to be combined must contain such a table. Other object
files may or may not contain it.

## Data Types (Not dices; portuguese joke)

Here are the types used for representing data in the ELF file objects.

{% include imagehb.html url="/images/blog/unbuilding-software-2/ELF.Data.Representation.png" caption="ELF Structure" %}

I build the
[elfdatatypes](https://github.com/geyslan/hb/blob/master/desconstruindo/elfdatatypes.c)
to show the types size in both architectures.

##ELF Header (Game Master)

There are two types of ELF header in /usr/include/elf.h: Elf32_Ehdr and
Elf64_Ehdr.

Here is the structure in C.

{% highlight c %}
#define EI_NIDENT 16

typedef struct {
   unsigned char  e_ident[EI_NIDENT]; /* Magic number and other info */
   ElfN_Half      e_type;             /* Object file type */
   ElfN_Half      e_machine;          /* Architecture */
   ElfN_Word      e_version;          /* Object file version */
   ElfN_Addr      e_entry;            /* Entry point virtual address */
   ElfN_Off       e_phoff;            /* Program header table file offset */
   ElfN_Off       e_shoff;            /* Section header table file offset */
   ElfN_Word      e_flags;            /* Processor-specific flags */
   ElfN_Half      e_ehsize;           /* ELF header size in bytes */
   ElfN_Half      e_phentsize;        /* Program header table entry size */
   ElfN_Half      e_phnum;            /* Program header table entry count */
   ElfN_Half      e_shentsize;        /* Section header table entry size */
   ElfN_Half      e_shnum;            /* Section header table entry count */
   ElfN_Half      e_shstrndx;         /* Section header string table index */
} ElfN_Ehdr;
{% endhighlight %}

Let's check the program1 ELF Header size.

{% highlight console %}
$ readelf -h programa1 | grep this
  Size of this header:               52 (bytes)
{% endhighlight %}

The ELF Header position, as mentioned, always will be at the object
file zero offset, but its size will depend on the architecture: 32
bits (52 bytes); 64 bits (64 bytes).

Let's look at the 52 bytes of the 32 bits ELF Header.

{% highlight console %}
$ xxd -l 52 programa1
0000000: 7f45 4c46 0101 0100 0000 0000 0000 0000  .ELF............
0000010: 0200 0300 0100 0000 0083 0408 3400 0000  ............4...
0000020: f007 0000 0000 0000 3400 2000 0800 2800  ........4. ...(.
0000030: 1e00 1b00                                ....
{% endhighlight %}


In accordance with the Elf32_Ehdr structure, the first 16 bytes are
concerning the identification of the object file (e_ident[16]). If we
add the size of the following types (Elf32_Half e_type [2 bytes],
Elf32_Half e_machine [2 bytes] and Elf32_Word e_version [4 bytes]),
we'll reach the offset 24 that refers to the entry point (0083 0408).

{% highlight console %}
$ readelf -h programa1 | grep Entry
  Entry point address:               0x8048300 (00830408 in inverse order)
{% endhighlight %}

Before finalizing, for not getting any disappointment, see a code in C
([elfentry]
(https://github.com/geyslan/hb/blob/master/desconstruindo/elfentry.c))
that reads the Entry Point value of programa2 object file.

{% highlight console %}
$ gcc -m32 elfentry.c -o elfentry
$ ./elfentry
Entry Point: 0x8048080

$ readelf -h programa2 | grep Entry
  Entry point address:               0x8048080
{% endhighlight %}

We're done here.</s>

Till next!  o/

{% include imagehb.html url="/images/blog/unbuilding-software-2/green_magic_by_katemaxpaint-d5qr63m.jpg" caption="Green Magic by KateMaxpaint" caption-link="http://katemaxpaint.deviantart.com/art/Green-Magic-347268514" %}

## More Info

[ELF - Wikipedia](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format)<br>
[Elf (another kind) - Wikipedia](http://en.wikipedia.org/wiki/Elf)<br>
[ABI - System V Application Binary Interface - SCO](http://www.sco.com/developers/devspecs/gabi41.pdf)<br>
[The ELF Object File Format - Introduction - Linux Journal](http://www.linuxjournal.com/article/1059)<br>
[The ELF Object File Format by Dissection - Linux Journal](http://www.linuxjournal.com/article/1060)<br>
[Dissecando ELF - Felipe Pena](http://www.mentebinaria.com.br/zine/edicoes/1/DissecandoELF.txt)<br>
[Linker (computing) - Wikipedia](https://en.wikipedia.org/wiki/Linker_(computing)%E2%80%8E)<br>
[Relocation (computing) - Wikipedia](http://en.wikipedia.org/wiki/Relocation_(computing)%E2%80%8E)<br>
[Loader (computing)](http://en.wikipedia.org/wiki/Loader_(computing)%E2%80%8E)<br>
