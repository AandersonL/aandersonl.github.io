---
layout: single

title: "Manipulating elf files in C++ using felf"
date: 2020-11-30 15:00:36 -0300
permalink: /:categories/:title/
categories: [programming, cybersec, linux, c++]

images_prefix: /assets/images/felflib/
---

A couple months ago I created [felf](https://github.com/AandersonL/felf), a library to parse [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) files into C++ structures, the reason for this was have a C++ way to work in ELF files using [STL](https://www.geeksforgeeks.org/the-c-standard-template-library-stl/) structures like vector, unordered maps and so on.

For this reason I wanna explore what was built and why, and show to you all the possibilities of this tiny yet nice library.

# Executable files

## What's this ?

This files is in your life more than never, you can find that in your SmartTV, Videogames, IoT devices and Unix systems.

The idea of this kind files knowns as ***executable files*** its to design a way to pack every information of a software into a single file that your OS will read, and do all the dirty work to map into memory every piece of code and resource that your CPU will use to execute.

ELF it's not the only one, [PE](https://en.wikipedia.org/wiki/Portable_Executable) is mainly used in Windows, [Mach-o](https://en.wikipedia.org/wiki/Mach-O) is used in MacOS systems, and for that article and library I will ***focus in Linux machines***, althouhg all the knowledge is the same that you need to work with PE and Mach-O files.

## From disk to memory

Imagine ELF files as a pre-fabricated home, the way the file is in disk contains almost the same structure that will be mapped in memory and in the same file contain all the information that OS has to follow to do everything in the corret way.


The structure is the follow:

![]({{site.url}}{{page.images_prefix}}elf.svg){:height="50%" width="50%"}


Everything is pretty straight forward:

* A ELF header that holds the basic information of the file
* A program header table that describe each segment of code that will be mapped
* Sections that hold some data like, executable code, string table, symbol table and so on
* Section header which is a array like structure that hold all sections information

# ELF Internal structures

Each of this structure are defined in ***elf.h*** and if you run ***man elf*** you will get the full documentation to work with that. Let's start by parsing in ELF header from disk (without loading anything in memory yet).

```c
#define EI_NIDENT 16

typedef struct {
    unsigned char e_ident[EI_NIDENT];
    uint16_t      e_type;
    uint16_t      e_machine;
    uint32_t      e_version;
    ElfN_Addr     e_entry;
    ElfN_Off      e_phoff;
    ElfN_Off      e_shoff;
    uint32_t      e_flags;
    uint16_t      e_ehsize;
    uint16_t      e_phentsize;
    uint16_t      e_phnum;
    uint16_t      e_shentsize;
    uint16_t      e_shnum;
    uint16_t      e_shstrndx;
} ElfN_Ehdr;
```

This struct specifies a block with of memory in the first bytes of the file, notice that we have a ***N*** in the variable name that can be used with ***32*** or ***64*** bits depending the OS and the file itself, the ELF header size can be 52 bytes in 32-bit files and 64 bytes in 64-bits files, you can verify that by looking the structs sizes:

```c
#include <elf.h>
...
printf("%d bytes\n", sizeof(Elf32_Ehdr)); // 52 bytes
printf("%d bytes\n", sizeof(Elf64_Ehdr)); // 64 bytes
```

## Parsing the header from scratch
As my system currently are in 64 bits, I will first dump out the first 64 bytes of data from my disk and use [BlobToChar](https://github.com/AandersonL/BlobToChar) to built a C array code with the header, with this dump I can load in my code and parse that quickly.


First 64 bytes (Header):

```
$ hexdump -C -n64 /usr/bin/ls                                                                                                   
00000000  7f 45 4c 46 02 01 01 00  00 00 00 00 00 00 00 00  |.ELF............|
00000010  03 00 3e 00 01 00 00 00  20 5b 00 00 00 00 00 00  |..>..... [......|
00000020  40 00 00 00 00 00 00 00  b0 23 02 00 00 00 00 00  |@........#......|
00000030  00 00 00 00 40 00 38 00  0b 00 40 00 1b 00 1a 00  |....@.8...@.....|
00000040
```
Dumping on disk:
```
$ dd if=/usr/bin/ls of=lselfheader bs=1 count=64                                                                           [130]
64+0 records in
64+0 records out
64 bytes copied, 0.000510657 s, 125 kB/s
```

Load in code in C arrays:

```
$ BlobToChar --blobname lselfheader                                                                                             
unsigned char buff[] = {0x7f,0x45,0x4c,0x46,0x2,0x1,0x1,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x3,0x0,0x3e,0x0,0x1,0x0,0x0,0x0,0x20,0x5b,0x0,0x0,0x0,0x0,0x0,0x0,0x40,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xb0,0x23,0x2,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x40,0x0,0x38,0x0,0xb,0x0,0x40,0x0,0x1b,0x0,0x1a,0x0,};
unsigned int buff_size = 64;
```

The parse code can ben written as:

```c
#include <elf.h>
#include <stdio.h>

unsigned char buff[] = {0x7f,0x45,0x4c,0x46,0x2,0x1,0x1,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x3,0x0,0x3e,0x0,0x1,0x0,0x0,0x0,0x20,0x5b,0x0,0x0,0x0,0x0,0x0,0x0,0x40,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0xb0,0x23,0x2,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x0,0x40,0x0,0x38,0x0,0xb,0x0,0x40,0x0,0x1b,0x0,0x1a,0x0,};
unsigned int buff_size = 64;

int main(int argc, char** argv)
{
	Elf64_Ehdr* elfHeader = (Elf64_Ehdr*) buff;
	printf("e_ident: %s\n", elfHeader->e_ident);
}
```

As structs are just aligned bytes in memory, we can use the struct ***Elf64_Ehdr*** to parse this raw bytes, with this we can access the elf header internal struct, in the above example I printed the ***e_indent***, which is an array with some basic information with the file itself, this are the first 16 bytes of this ***buf*** variable, and the very first 4 bytes: ***0x7f,0x45,0x4c,0x46*** contains the 'ELF' string starting from the second position.

```c
~ >>> ./parseheader                                                                              
Magic: ELF
```
Knowing that, we can now parse the whole file by reading the file itself and use the ELF structs to extract the executable data itself.

## Example: Patching sections names 

Let's make a cool example, let's use a change the ***.text*** section name to another thing.

In order to make that possible, we need to have access to the string table struct, which hold a array of strings with the ***0x00*** byte as delimiter.


![]({{site.url}}{{page.images_prefix}}string_table.png)

As the string table is just one portion of data in the file, it's defined as a normal section and the struct in x64 elf is defined as:


```c
typedef struct {
    uint32_t   sh_name;
    uint32_t   sh_type;
    uint64_t   sh_flags;
    Elf64_Addr sh_addr;
    Elf64_Off  sh_offset;
    uint64_t   sh_size;
    uint32_t   sh_link;
    uint32_t   sh_info;
    uint64_t   sh_addralign;
    uint64_t   sh_entsize;
} Elf64_Shdr;
```



With this struct in our hands, we just need to get the index of the section name (sh_name) and change by something else, notice that if we want to make the name greater or less then the real one, we will have to reshape this array and change all the sections and address information to maintain the file integrity, in order to make this simple as possible I will just change the ***.text*** name to ***.etxt***.

### Loading the entire file in memory

Before we continue, let's make a real code for this task and for that we need to load the file data in our memory, just like a loader will do.

For map files in memory in Linux, we will use the function [mmap](https://www.man7.org/linux/man-pages/man2/mmap.2.html) that will create a in ***memory mapping*** to a given [file descriptor](https://en.wikipedia.org/wiki/File_descriptor).

Function definition:

```c
 #include <sys/mman.h>

void *mmap(void *addr, size_t len, int prot, int flags, 
            int fildes, off_t off);
```

Using this idea, take a look in the following code to load our ELF file and dump the header, we will use this piece of code for the example part:

```c
#include <elf.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <stdio.h>



int main(int argc, char** argv)
{
	
	if (argc != 2) {
		fprintf(stderr, "Usage: %s <elf_path>\n", argv[0]);
		return 1;
	}

	const char* elf_path = argv[1];
	int file_fd = open(elf_path, O_RDWR); // Open in read and write

	if (!file_fd) return 1;
	
	struct stat st; // Get file size using stat function
	stat(elf_path, &st);
	unsigned file_size = st.st_size;

	// Map the current file descritor in any address in ours memory maps with size <file_size>
	void* addr = mmap(NULL, file_size, O_RDWR, MAP_SHARED, file_fd, 0);
	
	if (!addr) return 1;
		
	
	// Now we can work the new memory space using the raw address

	Elf64_Ehdr* elf_header = (Elf64_Ehdr*) addr;

	puts("elf->e_indent: ");

	for (int i = 0; i < EI_NIDENT; ++i) {
		printf("\t0x%x(%c)\n", elf_header->e_ident[i], elf_header->e_ident[i]);
	}

	// Unmap the values	
	munmap(addr,file_size);

	// Close the file descriptor
	close(file_fd);


	return 0;
}
```


If you compile the code above, you will get something like:

```
elf->e_indent: 
	0x7f()
	0x45(E)
	0x4c(L)
	0x46(F)
	0x2()
    ...
```

### Finding the section string table

Ok, now let's start the real job to get the section ***.text*** his name, In order to accomplish that we need the the first entry in the section array structure and get the total bytes used by this array, all this informations can be found in the elf header in the following fields:

* e_shnum - Holds the number of sections that our ELF file has
* e_shentsize - Holds the total raw size of each section
* e_shoff - Holds the offset of the first entry in the array

Knowing all that, we can calculate where the section array starts by getting the address of the mapped file + the ***e_shoff*** and then create a ***for*** loop where each "jump" is the index * e_shentsize, that way we can jump in each element of the array, after reach the .text section we can get where in the string table his name is defined.

```c
Elf64_Ehdr* elf_header = (Elf64_Ehdr*) addr;
uint16_t num_sections = elf_header->e_shnum;
uint16_t section_size = elf_header->e_shentsize;
Elf64_Off section_entry_offset = elf_header->e_shoff;

printf("%d Sections, with %d bytes each and starting at address 0x%x\n", num_sections, section_size, (uint64_t) addr + section_entry_offset);
```

>25 Sections, with 64 bytes each and starting at address 0x82702f68

Your numbers might differ based in the file that are you using and the mapped address that is used to calculate the entry of the array.

Now, we need to find the string table that will be used as an array, for our lucky index string table in the section array iseasily found in the ELF header in the field ***e_shstrndx***, to find the address using this index we just need to get the address of the first entry in the array and multiple the index with the size of each section.

pseudo-code:
```
string_table_address = (index * section_size) + first_entry_address
or
string_table_address = section_size) + section
```
Or even better, we can just get the first section address and just add the ***section_size***, if you are familiar in how array really works in memory, this will be easy to understand.

C code:
```c
Elf64_Shdr* string_table_section = (Elf64_Shdr*) ( (uint64_t) section + ((uint64_t) section_size * elf_header->e_shstrndx));
```

### Finding the .text string index
Now with the section header loaded, we can find the offset where the section data is stored, as this is only the header and the real content is in another place in the file, this data location is found in the field ***sh_offset*** in the section header, so in order to find the array we just need to get the ***mapped file address + section->sh_offset***.

After find the raw data in the file we just need to work how it's is specified in the ELF specs, this is a normal C-String array with the ***\00*** as delimiter.

C code to find the first entry in string table:

```c
section_name = (char*) (( (uint64_t) string_table_section->sh_offset + addr) + 1);
```

This will access the string array and get the first element, in order to pick the sections name we need access the field ***sh_name*** in the section header that holds the name index of this section in the string array, using that we can iterate in each section of the file and get each name easily, check the code:

```c
Elf64_Shdr* section = (Elf64_Shdr*) ((uint64_t) addr + section_entry_offset);	
Elf64_Shdr* string_table_section = (Elf64_Shdr*) ( (uint64_t) section + ((uint64_t) section_size * elf_header->e_shstrndx));

char* section_name;

for (int i = 0; i < num_sections; ++i) {
    section = (Elf64_Shdr*) ((uint64_t) section + section_size);
    section_name = (char*) (( (uint64_t) string_table_section->sh_offset + addr) + section->sh_name);
}
```

### Changing the .text name

Now, it's pretty simple, we can modify the name directly and as this file is mapped in memory in read-write mode our changes will be flushed direct in the diskfile, take a look:

```c	
for (int i = 0; i < num_sections; ++i) {
    section = (Elf64_Shdr*) ((uint64_t) section + section_size);
    section_name = (char*) (( (uint64_t) string_table_section->sh_offset + addr) + section->sh_name);
    if (!strcmp(section_name, ".text")) {
        strcpy(section_name, ".txet");
    }

}
```

Now take a look using ***readelf*** command to get all sections name:


![]({{site.url}}{{page.images_prefix}}text_patch.png)



The program still works because the new name is following all the elf specs, and it's a valid one.

# Enter felf

## Why felf

## Installing

## Using felf to simple operations

## Cool usages

### elf disassembly

### Section hashing


