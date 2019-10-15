# Chap1
## Preprocess and omit debug info; inspect output
Compile the the source code, stopping after preprocessing and omit debug info. 
Technically, the output of the below is not "compiled" but preprocessed code

```gcc -E -p compilation_example.c```

This essentialy copies all all headers included into the file, and replaces
macros with their constants. Note the example below
``` int main(int argc, char *argv[]) {
  printf(FORMAT_STR, MSG);
  return 0;
}
# 6 "compilation_example.c"
int main(int argc, char *argv[]) {
  printf("%s", "Hello, JOKER!\n");
  return 0;
}
```

## Compile the code to assembly
We can tell gcc to compile the source code to assembly code, and we can specify
the syntax to use; in this case we'll make it use intel. the default is AT&T
```gcc -S -masm=intel compilation_example.c```

The output of the above will be a .s file; common extension for assembly files.
The output is also very brief and readable as everything has been labeled; this
is not the case with stripped binaries.

## Assemble the code to an object file

Assembly phase input: assembly files ; output: object files. Typically, source, 
assembly, and object files have 1:1:1 relationship. We can generate an object
file with the -c flag

```gcc -c compilation_example.c```

The output of the above is a a .o object file, again relatively small.. note the ELF header
```
root@box:~/# file compilation_example.o

compilation_example.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped

root@box:~/# xxd compilation_example.o | head
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
00000010: 0100 3e00 0100 0000 0000 0000 0000 0000  ..>.............
00000020: 0000 0000 0000 0000 d802 0000 0000 0000  ................
00000030: 0000 0000 4000 0000 0000 4000 0d00 0c00  ....@.....@.....
00000040: 5548 89e5 4883 ec10 897d fc48 8975 f048  UH..H....}.H.u.H
00000050: 8d3d 0000 0000 e800 0000 00b8 0000 0000  .=..............
00000060: c9c3 4865 6c6c 6f2c 204a 4f4b 4552 2100  ..Hello, JOKER!.
00000070: 0047 4343 3a20 2844 6562 6961 6e20 392e  .GCC: (Debian 9.
00000080: 322e 312d 3429 2039 2e32 2e31 2032 3031  2.1-4) 9.2.1 201
00000090: 3930 3832 3100 0000 1400 0000 0000 0000  90821...........
```
Also note that the file is "relocatable" meaning it can be anywhere in memory
without breaking assumptions of the code; this differentiates the object file
from an executable file. Object files need to be relocatable because the assembler
has no way of knowing the addresses of other object files when assembling object
files.

## Linking
Last stage in compilation; links all object files into single binary. Linker
performs the linking. Object files may reference funcs / vars from other object
files or in extern libs. Pre linking, addresses where referenced code and data
will be placed is not known, so obj files only contain relocation symbols that
specify how to get funcs/vars. 

References that depend on a relocation symbol are called symbolic references. 
Object files may reference their own funcs/vars by absolute address, and this
reference will be symbolic as well. 

References to libs may/may not be resolved depending on the type of lib. Static
libraries with extension .a are merged into the binary, helping resolution. 
Dynamic libraries on the other hand are shared in mem among all programs on a 
given system, so instead of copying the lib into every program, the lib can
be loaded into memory 1 time and shared by other programs.

During linking, addresses of dynamic libs are not yet known and thus can't be
resolved. 

A complete binary can be compiled and execution-ready  by simply using gcc on 
the source code file
```gcc compilation_example.c```

## Symbols
Compilers emit symbols to keep track of symbolic names in binary code and data; 
such info is usually used by the linker. These symbols can be read from a binary
with readelf
```readelf --syms a.out```

This can be useful in a anumber of ways; more later. Below is an example.
```
    55: 0000000000002000     4 OBJECT  GLOBAL DEFAULT   16 _IO_stdin_used
    56: 0000000000001160    93 FUNC    GLOBAL DEFAULT   14 __libc_csu_init
    57: 0000000000004038     0 NOTYPE  GLOBAL DEFAULT   25 _end
    58: 0000000000001050    43 FUNC    GLOBAL DEFAULT   14 _start
    59: 0000000000004030     0 NOTYPE  GLOBAL DEFAULT   25 __bss_start
    60: 0000000000001135    34 FUNC    GLOBAL DEFAULT   14 main
```
Note: there are different fromats of debugging symbols; generally ELF -> DWARF
and PE -> Microsoft Portable Debugging (PDB) format. ELF includes symbols, 
PE's include PDB symbols in separate symbol file.


## Stripping binaries
Production ready code will be stripped, as will malware, which makes reversing
much more difficult. We can strip symbols from a binary with the strip
command. 

```strip --strip-all a.out```

Unsurprisingly, stripping i.e removing stuff, decreases the file size.

```
root@box:~/gitstuff/pba-notes/chap1# ll a.out a.out.bak
-rwxr-xr-x 1 root root 14416 Oct 13 18:19 a.out
-rwxr-xr-x 1 root root 16624 Oct 13 18:19 a.out.bak
```
Note our output of readelf is much smaller now...

```
root@box:~/gitstuff/pba-notes/chap1# cat syms-stripped

Symbol table '.dynsym' contains 7 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (2)
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.2.5 (2)
     4: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     5: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
     6: 0000000000000000     0 FUNC    WEAK   DEFAULT  UND __cxa_finalize@GLIBC_2.2.5 (2)
```
The difference is also very apparent when hexxdumping and comparing the two binaries
