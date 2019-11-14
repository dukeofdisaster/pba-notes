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

## Disassembling a binary
objdump utility can be used for all the disassembling. 

```
root@box:~/gitstuff/pba-notes/chap1# objdump -sj .rodata compilation_example.o

compilation_example.o:     file format elf64-x86-64

Contents of section .rodata:
 0000 48656c6c 6f2c204a 4f4b4552 2100      Hello, JOKER!.
root@box:~/gitstuff/pba-notes/chap1# objdump -M intel -d compilation_example.o

compilation_example.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:	55                   	push   rbp
   1:	48 89 e5             	mov    rbp,rsp
   4:	48 83 ec 10          	sub    rsp,0x10
   8:	89 7d fc             	mov    DWORD PTR [rbp-0x4],edi
   b:	48 89 75 f0          	mov    QWORD PTR [rbp-0x10],rsi
   f:	48 8d 3d 00 00 00 00 	lea    rdi,[rip+0x0]        # 16 <main+0x16>
  16:	e8 00 00 00 00       	call   1b <main+0x1b>
  1b:	b8 00 00 00 00       	mov    eax,0x0
  20:	c9                   	leave  
  21:	c3                   	ret    
root@box:~/gitstuff/pba-notes/chap1# 
```

Here -s == all data, -j == section, -M == disassembler options, -d == disassemble. 
The second command disassembles all the code in the object file in intel syntax. This is 
only the code of "main" function. Note the pointer to our hello string at line f: is 
set to rip+0x0. Note that the call immediately after points to 1b, which is oddly
the line below. This seems odd at first, but recall that object files contain 
data and code references that are not fully resolved because the compiler doesn't 
know at what base address the file will be loaded. Obj file waits for linker to
fill in the correct value of these references.

This can be confirmed with readelf to see all the relocation symbols in the
object file.

```
root@box:~/gitstuff/pba-notes/chap1# readelf --relocs compilation_example.o

Relocation section '.rela.text' at offset 0x228 contains 2 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000012  000500000002 R_X86_64_PC32     0000000000000000 .rodata - 4
000000000017  000b00000004 R_X86_64_PLT32    0000000000000000 puts - 4

Relocation section '.rela.eh_frame' at offset 0x258 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000020  000200000002 R_X86_64_PC32     0000000000000000 .text + 0
```
In .rela.text section, this symbol tells the linker to resolve the reference to
the string to point to whatever address it ends up at in .rodata. Below that,
this information tells the linker to resolve the call to puts. The left most 
column in the output above is the offest in the object file where the resolved
reference must be filled in. You'll also notice that it's equal to the offset 
of the instruction that needs to be fixed + 1; so at offset 0x12-1, we have opcode 3d
and at offset 0x17-1, we have opcode e8; the reolcations point to opcode+1 because
you don't want to overwrite the OPCODE, just the OPERAND, with the right data/address.

## examining a complete binary.
We can use objdump again to examine the disassembly of the compiled binary. To
see the differences, objdump the stripped and unstripped versions to a file and
diff them side-by-side with diff -y. See sample below
```

a.out:     file format elf64-x86-64			      |	a.out.bak:     file format elf64-x86-64


Disassembly of section .init:					Disassembly of section .init:

0000000000001000 <.init>:				      |	0000000000001000 <_init>:
    1000:	48 83 ec 08          	sub    rsp,0x8		    1000:	48 83 ec 08          	sub    rsp,0x8
    1004:	48 8b 05 dd 2f 00 00 	mov    rax,QWORD PTR  |	    1004:	48 8b 05 dd 2f 00 00 	mov    rax,QWORD PTR 
    100b:	48 85 c0             	test   rax,rax		    100b:	48 85 c0             	test   rax,rax
    100e:	74 02                	je     1012 <puts@plt |	    100e:	74 02                	je     1012 <_init+0x
    1010:	ff d0                	call   rax		    1010:	ff d0                	call   rax
    1012:	48 83 c4 08          	add    rsp,0x8		    1012:	48 83 c4 08          	add    rsp,0x8
    1016:	c3                   	ret    			    1016:	c3                   	ret    

Disassembly of section .plt:					Disassembly of section .plt:

0000000000001020 <puts@plt-0x10>:			      |	0000000000001020 <.plt>:
    1020:	ff 35 e2 2f 00 00    	push   QWORD PTR [rip |	    1020:	ff 35 e2 2f 00 00    	push   QWORD PTR [rip
    1026:	ff 25 e4 2f 00 00    	jmp    QWORD PTR [rip |	    1026:	ff 25 e4 2f 00 00    	jmp    QWORD PTR [rip
    102c:	0f 1f 40 00          	nop    DWORD PTR [rax	    102c:	0f 1f 40 00          	nop    DWORD PTR [rax

0000000000001030 <puts@plt>:					0000000000001030 <puts@plt>:
    1030:	ff 25 e2 2f 00 00    	jmp    QWORD PTR [rip |	    1030:	ff 25 e2 2f 00 00    	jmp    QWORD PTR [rip
    1036:	68 00 00 00 00       	push   0x0		    1036:	68 00 00 00 00       	push   0x0
    103b:	e9 e0 ff ff ff       	jmp    1020 <puts@plt |	    103b:	e9 e0 ff ff ff       	jmp    1020 <.plt>

Disassembly of section .plt.got:				Disassembly of section .plt.got:

0000000000001040 <__cxa_finalize@plt>:				0000000000001040 <__cxa_finalize@plt>:
    1040:	ff 25 b2 2f 00 00    	jmp    QWORD PTR [rip |	    1040:	ff 25 b2 2f 00 00    	jmp    QWORD PTR [rip
    1046:	66 90                	xchg   ax,ax		    1046:	66 90                	xchg   ax,ax

Disassembly of section .text:					Disassembly of section .text:

0000000000001050 <.text>:				      |	0000000000001050 <_start>:
    1050:	31 ed                	xor    ebp,ebp		    1050:	31 ed                	xor    ebp,ebp
    1052:	49 89 d1             	mov    r9,rdx		    1052:	49 89 d1             	mov    r9,rdx
    1055:	5e                   	pop    rsi		    1055:	5e                   	pop    rsi
    1056:	48 89 e2             	mov    rdx,rsp		    1056:	48 89 e2             	mov    rdx,rsp
    1059:	48 83 e4 f0          	and    rsp,0xffffffff	    1059:	48 83 e4 f0          	and    rsp,0xffffffff
    105d:	50                   	push   rax		    105d:	50                   	push   rax

```
While this output is kinda messy, the key take away is that in the stripped binary, 
our functions have been merged into one large blog in a section called .text, but
note that the disassembled instructions remain the same

## Loading and executing a binary
In-mem vs ondisk binaries won't match 1:1; some long 0 initialized ata may be
collapsed on disk. When starting a binary, OS allocates virtual address space
and a new process for the binary to run in. After, OS maps an interpreter into 
the process's virt memory. It's a user space program that knows how to load 
the binary and perform the necessary relocations. On linux, the intepreter
is usu. a shared lib called ld-linux.so. On windows, the interpreter is usually
the shared lib ntdll.dll. After loading the intepreter, kernel transfers control
to the interpreter.

Linux bins have section called .interp, that specifies the path to the intepreter
that is to be used to load the binary. We can see this with readelf

```
root@box:~/gitstuff/pba-notes/chap1# readelf -p .interp a.out

String dump of section '.interp':
  [     0]  /lib64/ld-linux-x86-64.so.2

root@box:~/gitstuff/pba-notes/chap1# readelf -p .interp a.out.bak

String dump of section '.interp':
  [     0]  /lib64/ld-linux-x86-64.so.2
```
Note there is no difference between the stripped/unstripped binaries in this 
section; after some consideration this seems intuitive, though we may be 
able to strip debugging symbols and function stubs, the OS still needs to
know what interpreter to use inorder to run the binary. 

After the intepreter loads the binary into its virt addr space (same as the 
interpreter). Then the interp parses the binary to find out which dynamic libs
the bin uses and maps these into the virt address space with mmap or an 
equivalent function and then performas necessary relocations to to fill in 
the binary's code sections with the right addresses. In reality, resolving 
references is deferred, not immediately done at load time. The references are
only resolved when they are invoked for the first time. Aka LAZY BINDING. 

After reloc, interpreter looks up the entry point of the binary and then 
transfers control to this entry point. 
