---
title: cprogram_generate
date: 2021-02-21 01:27:32
tags: compilelink
categories: c&cpp
---


### 关于编译链接执行，编译器和cpu相关
目标： 学完这个部分的知识和调试方法后，一方面可以在写程序时，减少编译错误等，在遇到编译和运行时错误，可以更快的解决或者知道怎么解决，用什么工具可以
更快的解决，另一方面，在处理cpu高载，dump问题时，能知道怎么处理和更好的处理； <!-- more -->
### 一个简单例子
说明linux下程序的预处理，编译，链接，运行的整个过程；
三个文件如下：
```cpp
main.cpp                               |func.cpp                               |func.h
#include<stdio.h>                      |#include "func.h"                      |int funcc(int a1);
#include<unistd.h>                     |int g_m = 3;                           |
#include<stdlib.h>                     |extern int g_n ;                       |~                                      
#include "func.h"                      |int funcc(int a1)                      |~                                      
extern int g_m;                        |{                                      |~                                      
int g_n = 3;                           |    int res = a1+3+g_n;                |~                                      
int main()                             |    return res;                        |~                                      
{                                      |}                                      |~                                      
    int g_x = 3;                       |                                       |~                                      
    printf("hello");                   |~                                      |~                                      
    setuid(234323);                    |~                                      |~                                      
    int c = funcc(g_x+2);              |~                                      |~                                      
    int *i = (int*)malloc(sizeof(int) *|~                                      |~                                      
2);                                    |~                                      |~                                      
    *i = 2;                            |~                                      |~                                      
    printf("%d  %lx\n",*i,i);          |~                                      |~                                      
    free(i);                           |~                                      |~                                      
    return 0;                          |~                                      |~                                      
}                                      |~                                      |~               
```
#### 各自预处理后：
gcc -E test.c -o test.i 或 gcc -E test.c
```cpp
func.i:
# 1 "func.cpp"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "func.cpp"
# 1 "func.h" 1
int funcc(int a1);
# 2 "func.cpp" 2
int g_m = 3;
extern int g_n ;
int funcc(int a1)
{
    int res = a1+3+g_n;
    return res;
}

main.i:
前面展开太多，不贴了：
extern void swab (const void *__restrict __from, void *__restrict __to,
    ssize_t __n) throw () __attribute__ ((__nonnull__ (1, 2)));
# 1151 "/usr/include/unistd.h" 3 4
}
# 3 "main.cpp" 2
# 1 "func.h" 1
int funcc(int a1);
# 4 "main.cpp" 2
extern int g_m;
int g_n = 3;                           
int main()                             
{                                      
    int g_x = 3;                       
    printf("hello");                   
    setuid(234323);                    
    int c = funcc(g_x+2);              
    int *i = (int*)malloc(sizeof(int) *2);                                    
    *i = 2;                            
    printf("%d  %lx\n",*i,i);          
    free(i);                           
    return 0;                          
}

```

#### 接着编译为汇编文件：
gcc -S main.cpp 
gcc -S func.cpp
从下面可以看到，已经对程序进行基本分段了，有数据段，代码段等，对全局变量，有专门的段来存取，对未确定的符号，用类型和等标识；
汇编语言和特定的cpu有关；
```cpp

        .file   "main.cpp"                                                |        .file   "func.cpp"
        .globl  g_n                                                       |        .globl  g_m
        .data                                                             |        .data
        .align 4                                                          |        .align 4
        .type   g_n, @object                                              |        .type   g_m, @object
        .size   g_n, 4                                                    |        .size   g_m, 4
g_n:                                                                      |g_m:
        .long   3                                                         |        .long   3
        .section        .rodata                                           |        .text
.LC0:                                                                     |        .globl  _Z5funcci
        .string "hello"                                                   |        .type   _Z5funcci, @function
.LC1:                                                                     |_Z5funcci:
        .string "%d  %lx\n"                                               |.LFB0:
        .text                                                             |        .cfi_startproc
        .globl  main                                                      |        pushq   %rbp
        .type   main, @function                                           |        .cfi_def_cfa_offset 16
main:                                                                     |        .cfi_offset 6, -16
.LFB2:                                                                    |        movq    %rsp, %rbp
        .cfi_startproc                                                    |        .cfi_def_cfa_register 6
        pushq   %rbp                                                      |        movl    %edi, -20(%rbp)
        .cfi_def_cfa_offset 16                                            |        movl    -20(%rbp), %eax
        .cfi_offset 6, -16                                                |        leal    3(%rax), %edx
        movq    %rsp, %rbp                                                |        movl    g_n(%rip), %eax
        .cfi_def_cfa_register 6                                           |        addl    %edx, %eax
        subq    $16, %rsp                                                 |        movl    %eax, -4(%rbp)
        movl    $3, -16(%rbp)                                             |        movl    -4(%rbp), %eax
        movl    $.LC0, %edi                                               |        popq    %rbp
        movl    $0, %eax                                                  |        .cfi_def_cfa 7, 8
        call    printf                                                    |        ret
        movl    $234323, %edi                                             |        .cfi_endproc
        call    setuid                                                    |.LFE0:
        movl    -16(%rbp), %eax                                           |        .size   _Z5funcci, .-_Z5funcci
        addl    $2, %eax                                                  |        .ident  "GCC: (Ubuntu 4.8.4-2ubuntu1~14.04.4) 4.8.4"
        movl    %eax, %edi                                                |        .section        .note.GNU-stack,"",@progbits
        call    _Z5funcci                                                  
        movl    %eax, -12(%rbp)                                           
        movl    $8, %eax
        movq    %rax, %rdi                                                
        call    malloc
        movq    %rax, -8(%rbp)                                            
        movq    -8(%rbp), %rax                                            
        movl    $2, (%rax)
        movq    -8(%rbp), %rax                                            
        movl    (%rax), %eax
        movq    -8(%rbp), %rdx                                            
        movl    %eax, %esi
        movl    $.LC1, %edi                                               
        movl    $0, %eax                                                  
        call    printf
        movq    -8(%rbp), %rax                                            
        movq    %rax, %rdi                                                
        call    free
        movl    $0, %eax                                                  
        leave
        .cfi_def_cfa 7, 8                                                 
        ret
        .cfi_endproc                                                      
.LFE2:  
        .size   main, .-main
        .ident  "GCC: (Ubuntu 4.8.4-2ubuntu1~14.04.4) 4.8.4"              
        .section        .note.GNU-stack,"",@progbits
                                                       
```
#### 编译为目标文件：
gcc -c main.cpp
gcc -c func.cpp
objdump -x func.o
```cpp
 file func.o
func.o: ELF 64-bit LSB  relocatable, x86-64, version 1 (SYSV), not stripped
为relocatable可重定向文件
objdump  -x func.o

func.o:     file format elf64-x86-64 文件格式
func.o
architecture: i386:x86-64, flags 0x00000011: 系统架构
HAS_RELOC, HAS_SYMS
start address 0x0000000000000000

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         0000001d  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000004  0000000000000000  0000000000000000  00000060  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  00000064  2**0
                  ALLOC
  3 .comment      0000002c  0000000000000000  0000000000000000  00000064  2**0
                  CONTENTS, READONLY
  4 .note.GNU-stack 00000000  0000000000000000  0000000000000000  00000090  2**0
                  CONTENTS, READONLY
  5 .eh_frame     00000038  0000000000000000  0000000000000000  00000090  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
SYMBOL TABLE:符号表，每个编译后的目标文件都有
0000000000000000 l    df *ABS*	0000000000000000 func.cpp
0000000000000000 l    d  .text	0000000000000000 .text
0000000000000000 l    d  .data	0000000000000000 .data
0000000000000000 l    d  .bss	0000000000000000 .bss
0000000000000000 l    d  .note.GNU-stack	0000000000000000 .note.GNU-stack
0000000000000000 l    d  .eh_frame	0000000000000000 .eh_frame
0000000000000000 l    d  .comment	0000000000000000 .comment
0000000000000000 g     O .data	0000000000000004 g_m
0000000000000000 g     F .text	000000000000001d _Z5funcci
0000000000000000         *UND*	0000000000000000 g_n  //依赖的外部，未定义的符号


RELOCATION RECORDS FOR [.text]:
OFFSET           TYPE              VALUE 
000000000000000f R_X86_64_PC32     g_n-0x0000000000000004


RELOCATION RECORDS FOR [.eh_frame]:
OFFSET           TYPE              VALUE 
0000000000000020 R_X86_64_PC32     .text

  file main.o
main.o: ELF 64-bit LSB  relocatable, x86-64, version 1 (SYSV), not stripped
 objdump  -x main.o

main.o:     file format elf64-x86-64
main.o
architecture: i386:x86-64, flags 0x00000011:
HAS_RELOC, HAS_SYMS
start address 0x0000000000000000

Sections:
Idx Name          Size      VMA               LMA               File off  Algn
  0 .text         00000081  0000000000000000  0000000000000000  00000040  2**0
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
  1 .data         00000004  0000000000000000  0000000000000000  000000c4  2**2
                  CONTENTS, ALLOC, LOAD, DATA
  2 .bss          00000000  0000000000000000  0000000000000000  000000c8  2**0
                  ALLOC
  3 .rodata       0000000f  0000000000000000  0000000000000000  000000c8  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .comment      0000002c  0000000000000000  0000000000000000  000000d7  2**0
                  CONTENTS, READONLY
  5 .note.GNU-stack 00000000  0000000000000000  0000000000000000  00000103  2**0
                  CONTENTS, READONLY
  6 .eh_frame     00000038  0000000000000000  0000000000000000  00000108  2**3
                  CONTENTS, ALLOC, LOAD, RELOC, READONLY, DATA
SYMBOL TABLE: 这个是符号表，即这个文件导出或者依赖的符号，用于模块间调用使用；
0000000000000000 l    df *ABS*	0000000000000000 main.cpp
0000000000000000 l    d  .text	0000000000000000 .text
0000000000000000 l    d  .data	0000000000000000 .data
0000000000000000 l    d  .bss	0000000000000000 .bss
0000000000000000 l    d  .rodata	0000000000000000 .rodata
0000000000000000 l    d  .note.GNU-stack	0000000000000000 .note.GNU-stack
0000000000000000 l    d  .eh_frame	0000000000000000 .eh_frame
0000000000000000 l    d  .comment	0000000000000000 .comment
0000000000000000 g     O .data	0000000000000004 g_n
0000000000000000 g     F .text	0000000000000081 main //下面为依赖的外部符号，有点多，可以看到，因为这里没有使用g_m，所以不依赖；
0000000000000000         *UND*	0000000000000000 printf
0000000000000000         *UND*	0000000000000000 setuid
0000000000000000         *UND*	0000000000000000 _Z5funcci
0000000000000000         *UND*	0000000000000000 malloc
0000000000000000         *UND*	0000000000000000 free


RELOCATION RECORDS FOR [.text]:
OFFSET           TYPE              VALUE 
0000000000000010 R_X86_64_32       .rodata
000000000000001a R_X86_64_PC32     printf-0x0000000000000004
0000000000000024 R_X86_64_PC32     setuid-0x0000000000000004
0000000000000031 R_X86_64_PC32     _Z5funcci-0x0000000000000004
0000000000000041 R_X86_64_PC32     malloc-0x0000000000000004
0000000000000060 R_X86_64_32       .rodata+0x0000000000000006
000000000000006a R_X86_64_PC32     printf-0x0000000000000004
0000000000000076 R_X86_64_PC32     free-0x0000000000000004


RELOCATION RECORDS FOR [.eh_frame]:
OFFSET           TYPE              VALUE 
0000000000000020 R_X86_64_PC32     .text

```
#### 接下来链接：
gcc -o main main.cpp  func.cpp 
也可以用ld命令链接，但是需要指定更多依赖；而gcc会自己去寻找一些默认的库等；

```cpp
 file main
main: ELF 64-bit LSB  executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=2110927a3d6ecf4dd0f70085d18bc49c3b61ba41, not stripped

```
可以看到这个文件为可执行文件类型；
通过elf工具看整个可执行程序的结构：上面的main.o func.o也是elf文件类型之一 
```cpp
 readelf -a main
 elf文件头：各个字段
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x400520
  Start of program headers:          64 (bytes into file)
  Start of section headers:          4544 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         30
  Section header string table index: 27

各个sections
Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00000238
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.ABI-tag     NOTE             0000000000400254  00000254
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .note.gnu.build-i NOTE             0000000000400274  00000274
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .gnu.hash         GNU_HASH         0000000000400298  00000298
       000000000000001c  0000000000000000   A       5     0     8
  [ 5] .dynsym           DYNSYM           00000000004002b8  000002b8
       00000000000000a8  0000000000000018   A       6     1     8
  [ 6] .dynstr           STRTAB           0000000000400360  00000360
       0000000000000052  0000000000000000   A       0     0     1
  [ 7] .gnu.version      VERSYM           00000000004003b2  000003b2
       000000000000000e  0000000000000002   A       5     0     2
  [ 8] .gnu.version_r    VERNEED          00000000004003c0  000003c0
       0000000000000020  0000000000000000   A       6     1     8
  [ 9] .rela.dyn         RELA             00000000004003e0  000003e0
       0000000000000018  0000000000000018   A       5     0     8
  [10] .rela.plt         RELA             00000000004003f8  000003f8
       0000000000000090  0000000000000018   A       5    12     8
  [11] .init             PROGBITS         0000000000400488  00000488
       000000000000001a  0000000000000000  AX       0     0     4
  [12] .plt              PROGBITS         00000000004004b0  000004b0
       0000000000000070  0000000000000010  AX       0     0     16
  [13] .text             PROGBITS         0000000000400520  00000520
       0000000000000202  0000000000000000  AX       0     0     16
  [14] .fini             PROGBITS         0000000000400724  00000724
       0000000000000009  0000000000000000  AX       0     0     4
  [15] .rodata           PROGBITS         0000000000400730  00000730
       0000000000000013  0000000000000000   A       0     0     4
  [16] .eh_frame_hdr     PROGBITS         0000000000400744  00000744
       000000000000003c  0000000000000000   A       0     0     4
  [17] .eh_frame         PROGBITS         0000000000400780  00000780
       0000000000000114  0000000000000000   A       0     0     8
  [18] .init_array       INIT_ARRAY       0000000000600e10  00000e10
       0000000000000008  0000000000000000  WA       0     0     8
  [19] .fini_array       FINI_ARRAY       0000000000600e18  00000e18
       0000000000000008  0000000000000000  WA       0     0     8
  [20] .jcr              PROGBITS         0000000000600e20  00000e20
       0000000000000008  0000000000000000  WA       0     0     8
  [21] .dynamic          DYNAMIC          0000000000600e28  00000e28
       00000000000001d0  0000000000000010  WA       6     0     8
  [22] .got              PROGBITS         0000000000600ff8  00000ff8
       0000000000000008  0000000000000008  WA       0     0     8
  [23] .got.plt          PROGBITS         0000000000601000  00001000
       0000000000000048  0000000000000008  WA       0     0     8
  [24] .data             PROGBITS         0000000000601048  00001048
       0000000000000018  0000000000000000  WA       0     0     8
  [25] .bss              NOBITS           0000000000601060  00001060
       0000000000000008  0000000000000000  WA       0     0     1
  [26] .comment          PROGBITS         0000000000000000  00001060
       0000000000000056  0000000000000001  MS       0     0     1
  [27] .shstrtab         STRTAB           0000000000000000  000010b6
       0000000000000108  0000000000000000           0     0     1
  [28] .symtab           SYMTAB           0000000000000000  00001940
       00000000000006c0  0000000000000018          29    46     8
  [29] .strtab           STRTAB           0000000000000000  00002000
       000000000000028f  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), l (large)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)

There are no section groups in this file.

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000400040 0x0000000000400040
                 0x00000000000001f8 0x00000000000001f8  R E    8
  INTERP         0x0000000000000238 0x0000000000400238 0x0000000000400238
                 0x000000000000001c 0x000000000000001c  R      1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x0000000000000894 0x0000000000000894  R E    200000
  LOAD           0x0000000000000e10 0x0000000000600e10 0x0000000000600e10
                 0x0000000000000250 0x0000000000000258  RW     200000
  DYNAMIC        0x0000000000000e28 0x0000000000600e28 0x0000000000600e28
                 0x00000000000001d0 0x00000000000001d0  RW     8
  NOTE           0x0000000000000254 0x0000000000400254 0x0000000000400254
                 0x0000000000000044 0x0000000000000044  R      4
  GNU_EH_FRAME   0x0000000000000744 0x0000000000400744 0x0000000000400744
                 0x000000000000003c 0x000000000000003c  R      4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     10
  GNU_RELRO      0x0000000000000e10 0x0000000000600e10 0x0000000000600e10
                 0x00000000000001f0 0x00000000000001f0  R      1

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .text .fini .rodata .eh_frame_hdr .eh_frame 
   03     .init_array .fini_array .jcr .dynamic .got .got.plt .data .bss 
   04     .dynamic 
   05     .note.ABI-tag .note.gnu.build-id 
   06     .eh_frame_hdr 
   07     
   08     .init_array .fini_array .jcr .dynamic .got 

Dynamic section at offset 0xe28 contains 24 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x000000000000000c (INIT)               0x400488
 0x000000000000000d (FINI)               0x400724
 0x0000000000000019 (INIT_ARRAY)         0x600e10
 0x000000000000001b (INIT_ARRAYSZ)       8 (bytes)
 0x000000000000001a (FINI_ARRAY)         0x600e18
 0x000000000000001c (FINI_ARRAYSZ)       8 (bytes)
 0x000000006ffffef5 (GNU_HASH)           0x400298
 0x0000000000000005 (STRTAB)             0x400360
 0x0000000000000006 (SYMTAB)             0x4002b8
 0x000000000000000a (STRSZ)              82 (bytes)
 0x000000000000000b (SYMENT)             24 (bytes)
 0x0000000000000015 (DEBUG)              0x0
 0x0000000000000003 (PLTGOT)             0x601000
 0x0000000000000002 (PLTRELSZ)           144 (bytes)
 0x0000000000000014 (PLTREL)             RELA
 0x0000000000000017 (JMPREL)             0x4003f8
 0x0000000000000007 (RELA)               0x4003e0
 0x0000000000000008 (RELASZ)             24 (bytes)
 0x0000000000000009 (RELAENT)            24 (bytes)
 0x000000006ffffffe (VERNEED)            0x4003c0
 0x000000006fffffff (VERNEEDNUM)         1
 0x000000006ffffff0 (VERSYM)             0x4003b2
 0x0000000000000000 (NULL)               0x0

Relocation section '.rela.dyn' at offset 0x3e0 contains 1 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000600ff8  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0

Relocation section '.rela.plt' at offset 0x3f8 contains 6 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000601018  000100000007 R_X86_64_JUMP_SLO 0000000000000000 free + 0
000000601020  000200000007 R_X86_64_JUMP_SLO 0000000000000000 printf + 0
000000601028  000300000007 R_X86_64_JUMP_SLO 0000000000000000 __libc_start_main + 0
000000601030  000400000007 R_X86_64_JUMP_SLO 0000000000000000 __gmon_start__ + 0
000000601038  000500000007 R_X86_64_JUMP_SLO 0000000000000000 malloc + 0
000000601040  000600000007 R_X86_64_JUMP_SLO 0000000000000000 setuid + 0

The decoding of unwind sections for machine type Advanced Micro Devices X86-64 is not currently supported.

Symbol table '.dynsym' contains 7 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND free@GLIBC_2.2.5 (2)
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@GLIBC_2.2.5 (2)
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@GLIBC_2.2.5 (2)
     4: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     5: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND malloc@GLIBC_2.2.5 (2)
     6: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND setuid@GLIBC_2.2.5 (2)

Symbol table '.symtab' contains 72 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000400238     0 SECTION LOCAL  DEFAULT    1 
     2: 0000000000400254     0 SECTION LOCAL  DEFAULT    2 
     3: 0000000000400274     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000400298     0 SECTION LOCAL  DEFAULT    4 
     5: 00000000004002b8     0 SECTION LOCAL  DEFAULT    5 
     6: 0000000000400360     0 SECTION LOCAL  DEFAULT    6 
     7: 00000000004003b2     0 SECTION LOCAL  DEFAULT    7 
     8: 00000000004003c0     0 SECTION LOCAL  DEFAULT    8 
     9: 00000000004003e0     0 SECTION LOCAL  DEFAULT    9 
    10: 00000000004003f8     0 SECTION LOCAL  DEFAULT   10 
    11: 0000000000400488     0 SECTION LOCAL  DEFAULT   11 
    12: 00000000004004b0     0 SECTION LOCAL  DEFAULT   12 
    13: 0000000000400520     0 SECTION LOCAL  DEFAULT   13 
    14: 0000000000400724     0 SECTION LOCAL  DEFAULT   14 
    15: 0000000000400730     0 SECTION LOCAL  DEFAULT   15 
    16: 0000000000400744     0 SECTION LOCAL  DEFAULT   16 
    17: 0000000000400780     0 SECTION LOCAL  DEFAULT   17 
    18: 0000000000600e10     0 SECTION LOCAL  DEFAULT   18 
    19: 0000000000600e18     0 SECTION LOCAL  DEFAULT   19 
    20: 0000000000600e20     0 SECTION LOCAL  DEFAULT   20 
    21: 0000000000600e28     0 SECTION LOCAL  DEFAULT   21 
    22: 0000000000600ff8     0 SECTION LOCAL  DEFAULT   22 
    23: 0000000000601000     0 SECTION LOCAL  DEFAULT   23 
    24: 0000000000601048     0 SECTION LOCAL  DEFAULT   24 
    25: 0000000000601060     0 SECTION LOCAL  DEFAULT   25 
    26: 0000000000000000     0 SECTION LOCAL  DEFAULT   26 
    27: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    28: 0000000000600e20     0 OBJECT  LOCAL  DEFAULT   20 __JCR_LIST__
    29: 0000000000400550     0 FUNC    LOCAL  DEFAULT   13 deregister_tm_clones
    30: 0000000000400580     0 FUNC    LOCAL  DEFAULT   13 register_tm_clones
    31: 00000000004005c0     0 FUNC    LOCAL  DEFAULT   13 __do_global_dtors_aux
    32: 0000000000601060     1 OBJECT  LOCAL  DEFAULT   25 completed.6982
    33: 0000000000600e18     0 OBJECT  LOCAL  DEFAULT   19 __do_global_dtors_aux_fin
    34: 00000000004005e0     0 FUNC    LOCAL  DEFAULT   13 frame_dummy
    35: 0000000000600e10     0 OBJECT  LOCAL  DEFAULT   18 __frame_dummy_init_array_
    36: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS main.cpp
    37: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS func.cpp
    38: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS crtstuff.c
    39: 0000000000400890     0 OBJECT  LOCAL  DEFAULT   17 __FRAME_END__
    40: 0000000000600e20     0 OBJECT  LOCAL  DEFAULT   20 __JCR_END__
    41: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS 
    42: 0000000000600e18     0 NOTYPE  LOCAL  DEFAULT   18 __init_array_end
    43: 0000000000600e28     0 OBJECT  LOCAL  DEFAULT   21 _DYNAMIC
    44: 0000000000600e10     0 NOTYPE  LOCAL  DEFAULT   18 __init_array_start
    45: 0000000000601000     0 OBJECT  LOCAL  DEFAULT   23 _GLOBAL_OFFSET_TABLE_
    46: 0000000000400720     2 FUNC    GLOBAL DEFAULT   13 __libc_csu_fini
    47: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND free@@GLIBC_2.2.5
    48: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
    49: 0000000000601048     0 NOTYPE  WEAK   DEFAULT   24 data_start
    50: 000000000040068e    29 FUNC    GLOBAL DEFAULT   13 _Z5funcci
    51: 0000000000601060     0 NOTYPE  GLOBAL DEFAULT   24 _edata
    52: 0000000000400724     0 FUNC    GLOBAL DEFAULT   14 _fini
    53: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND printf@@GLIBC_2.2.5
    54: 0000000000601058     4 OBJECT  GLOBAL DEFAULT   24 g_n
    55: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main@@GLIBC_
    56: 0000000000601048     0 NOTYPE  GLOBAL DEFAULT   24 __data_start
    57: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
    58: 0000000000601050     0 OBJECT  GLOBAL HIDDEN    24 __dso_handle
    59: 0000000000400730     4 OBJECT  GLOBAL DEFAULT   15 _IO_stdin_used
    60: 00000000004006b0   101 FUNC    GLOBAL DEFAULT   13 __libc_csu_init
    61: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND malloc@@GLIBC_2.2.5
    62: 0000000000601068     0 NOTYPE  GLOBAL DEFAULT   25 _end
    63: 0000000000400520     0 FUNC    GLOBAL DEFAULT   13 _start
    64: 0000000000601060     0 NOTYPE  GLOBAL DEFAULT   25 __bss_start
    65: 000000000040060d   129 FUNC    GLOBAL DEFAULT   13 main
    66: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _Jv_RegisterClasses
    67: 000000000060105c     4 OBJECT  GLOBAL DEFAULT   24 g_m
    68: 0000000000601060     0 OBJECT  GLOBAL HIDDEN    24 __TMC_END__
    69: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_registerTMCloneTable
    70: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND setuid@@GLIBC_2.2.5
    71: 0000000000400488     0 FUNC    GLOBAL DEFAULT   11 _init

Version symbols section '.gnu.version' contains 7 entries:
 Addr: 00000000004003b2  Offset: 0x0003b2  Link: 5 (.dynsym)
  000:   0 (*local*)       2 (GLIBC_2.2.5)   2 (GLIBC_2.2.5)   2 (GLIBC_2.2.5)
  004:   0 (*local*)       2 (GLIBC_2.2.5)   2 (GLIBC_2.2.5)

Version needs section '.gnu.version_r' contains 1 entries:
 Addr: 0x00000000004003c0  Offset: 0x0003c0  Link: 6 (.dynstr)
  000000: Version: 1  File: libc.so.6  Cnt: 1
  0x0010:   Name: GLIBC_2.2.5  Flags: none  Version: 2

Displaying notes found at file offset 0x00000254 with length 0x00000020:
  Owner                 Data size	Description
  GNU                  0x00000010	NT_GNU_ABI_TAG (ABI version tag)
    OS: Linux, ABI: 2.6.24

Displaying notes found at file offset 0x00000274 with length 0x00000024:
  Owner                 Data size	Description
  GNU                  0x00000014	NT_GNU_BUILD_ID (unique build ID bitstring)
    Build ID: 2110927a3d6ecf4dd0f70085d18bc49c3b61ba41
```
#### 接着来追踪下程序的运行：
```cpp
 ./main 
hello2  a61010
think@think-VirtualBox:~/c++/testld$ strace ./main 
execve("./main", ["./main"], [/* 74 vars */]) = 0
brk(0)                                  = 0x9ba000
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc78575f000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=86757, ...}) = 0
mmap(NULL, 86757, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7fc785749000
close(3)                                = 0
access("/etc/ld.so.nohwcap", F_OK)      = -1 ENOENT (No such file or directory)
open("/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0P \2\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=1840928, ...}) = 0
mmap(NULL, 3949248, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fc78517a000
mprotect(0x7fc785334000, 2097152, PROT_NONE) = 0
mmap(0x7fc785534000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1ba000) = 0x7fc785534000
mmap(0x7fc78553a000, 17088, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7fc78553a000
close(3)                                = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc785748000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc785746000
arch_prctl(ARCH_SET_FS, 0x7fc785746740) = 0
mprotect(0x7fc785534000, 16384, PROT_READ) = 0
mprotect(0x600000, 4096, PROT_READ)     = 0
mprotect(0x7fc785761000, 4096, PROT_READ) = 0
munmap(0x7fc785749000, 86757)           = 0
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 4), ...}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fc78575e000
setuid(234323)                          = -1 EPERM (Operation not permitted)
brk(0)                                  = 0x9ba000
brk(0x9db000)                           = 0x9db000
write(1, "hello2  9ba010\n", 15hello2  9ba010
)        = 15
exit_group(0)                           = ?
+++ exited with 0 +++
```
其实这里比较好的方式是gdb,单步调试，这个有兴趣再做吧；
运行的时候，是执行execve系列函数，将可执行文件加载后，转换为进程运行，这个时候，做的事情很多，需要创建新进程，需要将可执行文件映射到进程对应的内存区，并建立各个vma区域，对依赖的符号做库加载和映射，然后进程实例结构建立好后，根据进程需要的资源和当前运行状态，加入对应的进程相关队列，等待执行；

+ 至此，一个程序，从源文件，到可执行文件，到链接，转换为进程，运行结束，一个周期完结；

### 具体解释：预处理
1) 预处理阶段
c/c++是编译类型的语言，即c/c++源文件，是比较符合人类语言的文本文件，需要将这个文件翻译为机器能直接执行的可执行文件，这个翻译成可执行文件的过程，就是编译的过程；
程序在编译的过程中，需要经过几个阶段，第一个阶段是预处理阶段：
在正式的编译阶段之前进行。预处理阶段将根据已放置在文件中的预处理指令来修改源文件的内容。
如#include指令就是一个预处理指令，它把头文件的内容添加到.cpp文件中。
预处理指令，不是真正的程序语句，而是给预处理器的命令；
2) 预处理的指令：
预处理指令不止于#include,还有：
```cpp
#include
#define 
#ifdef
#end
#pragma
一些已定义好的宏：
__LINE__	整数值，表示当前正在编译的行在源文件中的行数。
__FILE__	字符串，表示被编译的源文件的文件名。
__DATE__	一个格式为 "Mmm dd yyyy" 的字符串，存储编译开始的日期。
__TIME__	一个格式为 "hh:mm:ss" 的字符串，存储编译开始的时间。
__cplusplus	整数值，所有C++编译器都定义了这个常量为某个值。如果这个编译器是完全遵守C++标准的，它的值应该等于或大于199711L，具体值取决于它遵守的是哪个版本的标准。
ref: 见手册；
```

预处理阶段，是预处理器对预处理指令的解析，并将解析后的内容输出出来或者到下个阶段的输入；
实践：
3) 例子：
```cpp
#include<stdio.h>
int main()
{
   printf("hello world\n");
   return 0;
}
 
gcc -E test.c -o test.i 或 gcc -E test.c  或用cpp
将它进行预处理：gcc -E simplest.c 可以看到没有引用到其他头文件：

# 1 "simplest.c"
# 1 "<built-in>"
# 1 "<command-line>"
# 1 "/usr/include/stdc-predef.h" 1 3 4
# 1 "<command-line>" 2
# 1 "simplest.c"
stdio.h的内容在这里铺开;
包含了各种函数的声明；
int main()
{
	printf("hello world\n");
    return 0;
}

```
其他c/cpp文件也是一样，但是一个程序(进程)只能有一个main函数
PS:关于g++和gcc的不同： g++ 比gcc 多了一些库的链接功能，当使用了c++的标准库的时候，必须用g++;

4) 说明：
预处理中的#include指令，在编译器处理后，会将对应的头文件展开，这样，其中的函数声明就能展开在引用头文件的源码文件中，当之后编译程序时，用到的符号，
声明就能被找到；


### 具体解释：编译
#### 编译阶段概述：
这个阶段，编译器会把每个实现文件，结合头文件编译成中间文件(其实编译过程是不需要头文件的，原因是一些文件中会引用头文件中的函数等符号，所以需要，而也可以引用.c文件，但是会出现重复定义，而声明是可以重复)，一般是一个二进制.o文件，并进行相关的编译器优化，即将一些语句顺序进行调节等；
先把源文件翻译成汇编文件，再通过汇编器将汇编文件翻译为二进制文件；

#### 一个例子：
```cpp
int m()
{
   int a = 3;
   return a;
}
```
gcc -S hh.cpp
生成hh.s:

```cpp
	.file	"hh.cpp"
	.text
	.globl	_Z1mv
	.type	_Z1mv, @function
_Z1mv:
.LFB0:
	.cfi_startproc
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset 6, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register 6
	movl	$3, -4(%rbp)
	movl	-4(%rbp), %eax
	popq	%rbp
	.cfi_def_cfa 7, 8
	ret
	.cfi_endproc
.LFE0:
	.size	_Z1mv, .-_Z1mv
	.ident	"GCC: (Ubuntu 4.8.4-2ubuntu1~14.04.4) 4.8.4"
	.section	.note.GNU-stack,"",@progbits
```
暂时不解析

#### 编译时不需要依赖的调用的实现
编译期间，都是用的-c生成各种.o文件，进而是静态库或动态库，或者单纯的有main函数的中间文件；
这个时候不需要在编译时提供依赖的实现文件，比如依赖的函数实现文件，只需要头文件即可生成；
```cpp
eg:
dep.h:
void def();

hello.cpp
#include "dep.h"

int hello()
{
    def();
    return 0;
}


hello.h
int hello();

main.cpp
#include "hello.h"
int main()
{
    hello();
    return 0;
}
```
编译：     
gcc -c hello.cpp

gcc -c main.cpp

生成可执行时才需要链接：这个时候需要所有的实现接口对应点的实现文件：或以库的形式，或以动态库的形式，或者实现文件.o/.cpp
```cpp
keshixing@ubuntu:~/testld$ g++ -o main main.cpp hello.o
/usr/bin/ld: hello.o: in function `hello()':
hello.cpp:(.text+0x9): undefined reference to `def()'
collect2: error: ld returned 1 exit status

```
这里没有实现def()所以报错了；

编译过程中汇编器将汇编文件翻译为.o中间文件，这里汇编器其实已经包含到gcc中了，也可以自己下载一个特定平台的比如：NASM
输出可执行程序时，必须实现入口函数，比如main;
gcc -o main main.s

#### 编译过程总结：
关于编译过程涉及如何将源程序转换为汇编程序，主要是编译器的部分工作，具体是词法分析，语法分析等，有著名的lex,yacc工具，也有
对应的状态机等逻辑，内容很多，有兴趣可以看编译原理 龙书；

这里，记录下：对编译后的程序单元，得到的是什么：是一种elf格式的文件，对linux来说；
elf文件，有以下几种类型：
可重定位文件： relocatable file .o文件
可执行文件：executable file
共享目标文件：shared object file ，通常是so动态库
dump文件： core-dump file 

所以编译的结果，是生成第一种elf目标文件；
而主要的内容：
```cpp
1 elf格式头
2 elf格式的各个段
3 elf各个段的内容，比如代码段，里面包括了编译后的代码等；
4 符号表；
5 。。。

其中最重要的，就是代码段和数据段，符号表等；
代码段，展示了这个目标文件的功能了流程，数据段，表示了存放的全局数据等，符号表，展示了这个目标文件提供了哪些可以被外部使用的符号，以及
依赖的哪些符号；以便链接使用
```
### 具体解释：链接
#### 基本概述：

+ 链接的来源：
写一个程序，需要定义各种函数，当只有一个源文件，编译为一个可执行程序时，这个时候，调用函数的地方直接填上函数的地址，看起来没什么问题；
但是当调整函数的定义时，不考虑其他因素，则函数的基地址在代码段中的位置，又会发生变化，这个时候，调用的位置有需要修改，这个是一个繁琐的工作；

于是这里提供了符号的方式，调用函数的地方，转换为符号，根据符号寻找对应的地址，这样，即使修改函数，也不用改调用的地方；

当程序越来越大，拆分为多个目标文件模块，这个时候，对外部符号的链接 ，有两种形式：
一种是在产生可执行程序，链接时，将外部符号对应的定义模块，结合
到调用的文件中，程序通过符号表和重定位等，转换为正确的地址；程序包含符号对应函数的定义，执行时可以直接运行，不依赖外部库；
这种形式是静态链接

另一种形式，在链接时，并不是将实现的代码结合进去，只是带符号，而在运行时，依赖对应的系统上的动态库；执行时再根据调用的符号，到对应
的系统上寻找动态库，加载到内存中，来执行；






+ 链接的进一步解释和相关命令：
关于编译时的链接：
我们在编译一个完整的程序时，除了指定含有main函数的程序文件外，这个文件还可能依赖定义在其他源程序中的函数，这个时候，编译时需要指定如：
gcc -o main main.cpp func.cpp ...
有时候这些源文件以编译好的.o的形式呈现，也可以进行：gcc -o main main.cpp func.o
而当许多.o需要指定比较麻烦，所以库出现了，用来将.o文件汇总，分为两种形式，一种是静态库，一种是动态库；
静态库，就是在编译时将文件的执行代码都聚合到可执行文件中，包括调用的功能函数等，执行时不需要额外的库支持；编译出来的可执行文件会比较大；
动态库，即在编译时只是将调用的函数符号指定到符号表中，在执行时才去系统默认库路径(头文件指定的)寻找库,并链接该函数执行；

静态链接：在编译时，要指定链接的静态库，包括位置等；
eg: 
gcc -o main main.cpp func.a /home/zhangsan/lib/funall.a local/local.a 
动态链接：编译时，需要指定链接的动态库，包括位置等；默认会找默认位置的动态库
gcc -o main main.cpp -L./xxx   -lcurl -lpthread 
-L./xx 为指定了目录下有动态链接库，也可以链接成功；
注意形式为：libxxx.so 链接为 -lxxx
在运行的时候：
ldd a.out这种可以得到可执行文件希望在哪里找到库；而一般要设置export LD_LIBRARY_PATH=/home/think/c++/ 或者放在
系统默认搜索路径下，或者修改系统文件/etc/ld.so.conf，添加路径，运行ldconfig命令
默认搜索路径：
/lib
/usr/lib
/usr/local/lib 



#### 静态链接
静态链接用到的技术：
+ 如何将模块.o文件合并到输出的可执行文件：地址和空间分配
需要考虑字节对齐等，每个模块文件的格式都是类似，有各个对应的段，这样可以将相似的段进行合并；
比如.text ->.text

+ 符号解析和重定位：
在将段合并后，各个符号对应的函数等(相对位置)已经确定了，这样可以进行符号解析，调整代码中的地址等；
1) 可以通过objdump -h xx.o 和可执行文件的对比，看到text段等段的相对偏移位置已经确定了；
2) 可以通过objdump -d xx.o 和objdump -d xxmain来对比链接前后，对外部符号如全局变量和函数的汇编语句中的地址数值，是不是
从0->真正的位置；
在elf文件中，有个 结构叫重定位表（relocation table),具体见elf格式解析，专门存放和重定位相关的信息；
.rel.xxx
3) 链接器会在链接时，去重定位表中找到对应符号的地址，然后进行符号解析，将符号改为正在的相对地址；
+ 相关命令：
1）链接的符号查看： nm xxx.o可以看到提供的符号和依赖的符号；
2） ld main.o func.o -e main -o abc -lc
3) ar -t libc.a 查看a文件中有哪些o文件；
4） gcc -static  --verbose -fno-builtin xx.c 输出详细信息，编译时；

+ 链接过程控制--链接脚本：
链接器提供了多种控制整个链接过程的方法：
1) 提供命令行来给链接器指定参数；
2) 将链接指令存放在目标文件中；比如vc中经常使用
3) 使用链接脚本：在使用ld时，使用ld -T xxx.script

一个例子说明链接脚本是做什么的：
https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_chapter/ld_3.html



### 具体解释：装载和动态链接
#### 概述：
进程和程序有什么不同？
#### 进程的虚拟地址空间

#### 程序装载是什么，装载有什么方式？
装载其实，就是将磁盘中的程序文件，按一定的格式，载入到内存中；这其中会用到页映射等操作系统对文件和内存管理的部分功能；
装载，只是程序转换为进程的其中一个步骤，一个程序，如何转换为一个进程？
1) 通过execvl系列函数，建立进程；创建一个独立的虚拟地址空间；
2) 读取可执行文件，将可执行文件映射到进程虚拟地址空间中，并建立各个段-vma 区域，一个vma由多个段组成；
3) 当调度器调度到此进程，将cpu指令寄存器设置为进程中可执行段的入口，启动运行
这之后，其实可执行程序并未全部都载入到内存中，在执行过程中，会出现缺页错误，然后操作系统进行处理，但是用户无感知；

#### 动态链接：
+ 为什么要动态链接：
如果用静态链接，可执行程序过大，副本多，占用磁盘大，不好传输，模块更新困难等等；
动态链接，是在执行程序的过程中发生的：具体来讲，发生在装载的时候；
+ 动态链接如何实现？
在linux中，elf动态链接文件叫共享对象 dynamic shared obj
动态链接的程序，在程序被装载时，系统的动态链接器会将程序需要的所有动态链接库装载到进程的地址空间，并将所有未决议的符号
绑定到相应的动态链接库中，并进行重定位； 注意动态链接器和静态链接器不同；

动态链接，会将链接和重定位的工作放到了运行时，这样会导致程序性能下降，但动态链接器也做了一些优化，比如延迟绑定等；
+ 如何生成动态库：
gcc -fPIC -shared -o xx.so xx.c

+ 运行时查看进程的各个段映射：
cat /proc/pid/maps


+ 动态链接库中的代码可以被多个程序共享，由此带来重定位的问题：
1) 原因：
动态库被装载时，共享对象，如何确定它在进程虚拟地址空间中的位置？ 装载时重定位，即类似静态链接，在模块装载地址确定时，系统对
程序中所有的绝对地址引用进行重定位。但是这个对不同进程来说，会产生不同的地址；所以同一份指令在被多个进程共享时会出现问题。无法做到共享省内存
的目的；这种方法其实就是-shared;

2) 进阶：使用-fPIC: 地址无关代码： position-independent-code
可以解决以上办法，但是只能有部分可以达到目的：部分指令在多个进程间是无法共享的，所以需要将这部分抽离处理和数据段放在一起；
对数据，函数引用来说有以下四种，只有两种可以：
(1) 模块内部的函数调用，跳转  相对位置固定，不需要重定位
(2) 模块内部的数据访问，比如模块中定义的全局变量，静态变量 相对位置固定，不需要重定位
(3) 模块外部函数的调用，跳转   依赖外部符号，需要将其放到数据段，使用全局偏移表 GOT
(4) 模块外的数据访问，比如用到了其他模块中定义的全局变量；需要用GOT表
使用-fPIC就是用来产生地址无关代码的

+ 延迟绑定


+ 动态链接相关结构：
1）.interp段
2) .dynamic段
3) 动态链接符号表等；
4）动态链接重定位表


### 关于编译，链接形成的目标文件；
#### 关于目标文件：
在编译时除了可以生成可执行文件，还可以生产静态库，动态库，中间文件，汇编文件等，这里看静态库和动态库的生成规则
生成静态库文件：
ar rvs libfunc.a func.o func2.o
man ar 查看更多
生成动态库文件:
gcc func.cpp  -fPIC -shared -o libfunc.so
-shared 该选项指定生成动态连接库（让连接器生成T类型的导出符号表，有时候也生成弱连接W类型的导出符号），不用该标志外部程序无法连接
Produce a shared object which can then be linked with other objects to form an executable.  Not all systems support this option.  For predictable results, you must also specify the same

-fPIC：表示编译为位置独立的代码，不用此选项的话编译后的代码是位置相关的所以动态载入时是通过代码拷贝的方式来满足不同进程的需要，而不能达到真正代码段共享的目的。
更多见man gcc
5) 总：
一个完整的gcc/g++编译命令，除了指定上面的内容，还需要指定头文件的位置，默认目录有哪些？，优化选项，-c/-o/-S/-E其他选项等，这里把这些汇总下，并给上参考地址方便查阅；man gcc
-c:  -c  Compile or assemble the source files, but do not link.  The linking stage simply is not done.  The ultimate output is in the form of an object file for each source file.
-I: 指定头文件的路径
gcc -I. -I /home/include/ ..
指定了头文件路径后，在#include时可以省略路径；
#### elf格式
/usr/include/elf.h中包含了elf文件结构的各个字段格式；
https://refspecs.linuxfoundation.org/elf/elf.pdf
或者书籍： 程序员的自我修养等可以查阅到具体信息；
### 关于入口函数和运行时库CRT：
入口函数不一定是main，在main调用之前，为了程序能顺利运行，需要初始化程序的执行环节，如堆分配初始化等，以及c++中的
全局对象构造函数，都在main之前执行，而全局对象的析构函数在main之后被执行；
这个依赖于elf中的两种特殊的段： .init .finit
所以在链接后，程序最开始运行的，不是main，而是.init中指定的入口函数，它可能包括各种初始化，如运行时库的入口函数，c++全局对象的构造函数，等等；

+ 运行时库：
一个c运行时库大概包括如下功能：
1) 启动和退出，包括入口函数及其依赖
2) 标准库函数集合
3) IO的封装
4) 堆的分配等封装
5) 语言中一些特殊功能实现
6) 调试功能；

另外，链接脚本或参数也可以指定不是main作为主函数，而是用别的名字，如ld -e nomain
### 关键词：
ABI： 指的是不同的cpu架构，操作系统，编译器类型甚至版本带来的目标文件差异；
BFD库： 为了解决多种cpu，大小端下，可执行程序兼容性问题，提供了统一的格式；










### 其他：

##### c/c++的反汇编，二进制查看，可执行文件格式等；
1) 可执行文件的结构：
think@think-VirtualBox:~/c++$ size hh.o
   text	   data	    bss	    dec	    hex	filename
     72	      0	      0	     72	     48	hh.o
代码段：只读，可共享，代码中的常量数据在编译时在代码段中分配空间，如int x=3;中的3及const 声明的常量和字符串常量;还有就是代码本身的0101机器码
数据段：为全局初始化数据区和静态数据区：已经初始化的全局变量和静态变量（全局和局部）
bss:为初始化数据取，存放为初始化的全局变量和静态变量，但在开始执行时会被内核赋值０或null


2）如何反汇编；
objdump 工具，除了可以反汇编，还可以查看二进制文件等：
objdump -s hh.o : 查看十六进制
```cpp
think@think-VirtualBox:~/c++$ objdump -s  hh.o

hh.o:     file format elf64-x86-64

Contents of section .text:
 0000 554889e5 c745fc03 0000008b 45fc5dc3  UH...E......E.].
Contents of section .comment:
 0000 00474343 3a202855 62756e74 7520342e  .GCC: (Ubuntu 4.
 0010 382e342d 32756275 6e747531 7e31342e  8.4-2ubuntu1~14.
 0020 30342e34 2920342e 382e3400           04.4) 4.8.4.    
Contents of section .eh_frame:
 0000 14000000 00000000 017a5200 01781001  .........zR..x..
 0010 1b0c0708 90010000 1c000000 1c000000  ................
 0020 00000000 10000000 00410e10 8602430d  .........A....C.
 0030 064b0c07 08000000                    .K......        
 ```

 objdump -d hh.o: 反汇编：


### 运行时：
##### c/c++文件如何执行，动态链接如何进行；
c/c++可执行文件如何执行呢？
可执行文件分为完整的非动态链接的可执行文件和完整的静态链接的可执行文件
1）静态链接：在./a.out的时候，会通过触发起一个execl系列函数，起一个进程，并开始执行，执行过程即会创建进程空间，内核管理，内核分时间片运行；
即作为一个进程来运行，而进程有自己的生命周期：就绪，运行，阻塞等等，也有各种进程上下文，当前的资源等等；
总的来讲： 生命周期，进程管理的私有结构，进程的动态空间：堆，栈，虚拟地址空间，各种阻塞队列等等；
2）动态链接：动态链接的可执行文件，需要搜索路径下有依赖的动态库，否则无法运行，在运行时会通过链接的符号等去寻找对应的动态库函数实现，加载调用运行；
之后的运行和静态链接基本一致；

##### gdb基础和gdb常用，手册等；
当编译时带上-g后即带debug信息，运行时即可以进行gdb 运行；
gdb main
gdb attach 挂的是线程还是进程？
见gdb文章
#### c++程序执行过程调试例子，注意执行语句先后，编译器优化前后不同


#### 堆栈空间表现例子
堆栈空间如何增长呢？
跟cpu系统有关，暂时不深究；要知道new/malloc是在堆上分配空间，普通定义数组，变量是在栈上分配空间；
struct class这种的，定义一个非指针变量，是放栈还是堆？

#### 多线程相关调试例子；
1）查看多线程pid: top -Hp pid
2) 多线程是系统层面的，多线程的实现可以是用系统的接口，比如posix的pthread,fork进程等，或者用语言的thread ,这种最后也是调用的pthread系列函数；
这里默认都是unix系统；
3）使用linux系统工具调试多线程程序；

#### linux core-dump机制和相关工具
1) core-dump是什么？
2）core-dump如何触发？




