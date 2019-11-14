# ELF FORMAT
Focus on 64-bit format elf files; should be easy to generalize to 32-bit format.

Elf is used for: executables, objects, shared libs, and core dumps. 

At a high level, elf binaries consist of 4 types of components.

1. Executable header
2. Program headers
3. A number of sections
4. A series of optional section headers.

## Executable header
Every ELF file starts with an executable header; a structured series of bytes
saying whats in the ELF file, what kind of ELF file it is, and where in the
file to find all the other contents. To find out what format the ex.head. is,
you can look up its type def in /usr/include/elf.h or in the ELF specification
at http://refspecs.linuxbase.org/elf/elf.pdf. 

### Definition of ELF64_Ehdr in /usr/unclude/elf.h

```
typedef struct {
  unsigned char e_ident[16]; /* Magic number and other info       */
  uint16_t  e_type;          /* object file type                  */
  uint16_t  e_machine;       /* Architecture                      */
  uint16_t  e_version;       /* Object file Version               */
  uint16_t  e_entry;         /* Entry point virt address          */
  uint16_t  e_phoff;         /* Program header table file offset  */
  uint16_t  e_shoff;         /* Section header table file offset  */
  uint16_t  e_flags;         /* Processor-specific flags          */
  uint16_t  e_ehsize;        /* ELF header size in bytes          */
  uint16_t  e_phentsize;     /* Program header table entry size   */
  uint16_t  e_phnum;         /* Program header table entry count  */
  uint16_t  e_shentsize;     /* Section header table entry size   */
  uint16_t  e_shnum;         /* Section header table entry count  */
  uint16_t  e_shstrndx;      /* Section header string table index */
} ELF64_Ehdr;
```
