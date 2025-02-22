---
# User change
title: Example application

weight: 3 # 1 is first, 2 is second, etc.

# Do not modify
layout: "learningpathall"
---
We will create a simple `hello world` type application to demonstrate how pointer authentication works.

The example contains a function `func2()` to start a shell, that is never called by the application.

Using your preferred text editor, create `main.c` containing the below.

```C
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void func1(char *s)
{
    char buffer[16];
    strcpy(buffer, s);
}

void func2(void)
{
    printf("Hello from func2!\n");
    system("/bin/sh");
}

int main(int argc, char **argv)
{
    if (argc > 1)
    {
        func1(argv[1]);
        printf("Hello World!\n");
    }

    return 0;
}
```
Create a `Makefile` with the contents below.

```Makefile
CROSS_COMPILE ?= aarch64-linux-gnu-
CC=$(CROSS_COMPILE)gcc

CFLAGS += -march=armv8.5-a -fPIC -pedantic -Wall -Wextra -ggdb3 -O0
LDFLAGS += -fPIE -static

PROJ := main_pac
PROJ_NOPAC := main_nopac

.PHONY: all clean

all: none pac

clean:
	rm -f $(PROJ) $(PROJ_NOPAC)

none: CFLAGS+=-fno-stack-protector
none: $(PROJ_NOPAC)

pac: CFLAGS+=-mbranch-protection=standard -fno-stack-protector
pac: $(PROJ)

$(PROJ): main.c
	$(CC) $(CFLAGS) $(LDFLAGS) main.c -o $@

$(PROJ_NOPAC): main.c
	$(CC) $(CFLAGS) $(LDFLAGS) main.c -o $@

dump_none: $(PROJ_NOPAC)
	gdb-multiarch -batch -ex "disassemble main" $<
	gdb-multiarch -batch -ex "disassemble func1" $<
dump_pac: $(PROJ)
	gdb-multiarch -batch -ex "disassemble main" $<
	gdb-multiarch -batch -ex "disassemble func1" $<
```

## Understanding the compiler switches 

Arm Compiler for Embedded, GCC, and LLVM can all produce code making use of pointer authentication.

Generation of such code is controlled by:
```console
-mbranch-protection=<protection>
```
Where `<protection>` can have different inputs:
* `none` means that no branch protection enabled. This is the default.
* `standard` enables all types of PAC return address signing and branch protection.

## Build the application with and without PAC instructions 

Build with the command:

```console
make all
```

This will create 2 executables, `main_nopac` and `main_pac`.

| Application | -mbranch-protection |
|:-----------:|:-------------------:|
|  main_nopac | none                |
|  main_pac   | standard            |


## Inspect the disassembled instructions

Inspect the difference in the generated code used for applications built with and without pointer authentication and BTI.

### main_nopac

Inspect the disassembled contents of functions `main` and `func1` in `main_nopac`:
```console
make dump_none
```
You will see the similar to the following:
```output
Dump of assembler code for function main:
   0x0000000000400718 <+0>:     stp     x29, x30, [sp, #-32]!
   0x000000000040071c <+4>:     mov     x29, sp
   0x0000000000400720 <+8>:     str     w0, [sp, #28]
   0x0000000000400724 <+12>:    str     x1, [sp, #16]
   0x0000000000400728 <+16>:    ldr     w0, [sp, #28]
   0x000000000040072c <+20>:    cmp     w0, #0x1
   0x0000000000400730 <+24>:    b.le    0x400750 <main+56>
   0x0000000000400734 <+28>:    ldr     x0, [sp, #16]
   0x0000000000400738 <+32>:    add     x0, x0, #0x8
   0x000000000040073c <+36>:    ldr     x0, [x0]
   0x0000000000400740 <+40>:    bl      0x4006d4 <func1>
   0x0000000000400744 <+44>:    adrp    x0, 0x459000 <do_release_all+32>
   0x0000000000400748 <+48>:    add     x0, x0, #0x710
   0x000000000040074c <+52>:    bl      0x4076d0 <puts>
   0x0000000000400750 <+56>:    mov     w0, #0x0                        // #0
   0x0000000000400754 <+60>:    ldp     x29, x30, [sp], #32
   0x0000000000400758 <+64>:    ret
...
Dump of assembler code for function func1:
   0x00000000004006d4 <+0>:     stp     x29, x30, [sp, #-48]!
   0x00000000004006d8 <+4>:     mov     x29, sp
   0x00000000004006dc <+8>:     str     x0, [sp, #24]
   0x00000000004006e0 <+12>:    add     x0, sp, #0x20
   0x00000000004006e4 <+16>:    ldr     x1, [sp, #24]
   0x00000000004006e8 <+20>:    bl      0x415880 <strcpy>
   0x00000000004006ec <+24>:    nop
   0x00000000004006f0 <+28>:    ldp     x29, x30, [sp], #48
   0x00000000004006f4 <+32>:    ret
```
Both the functions in this case use the `stp` to push pairs of 64-bit registers onto the stack at entry, and `ret`to return from the functions.

### main_pac

Now let us inspect the disassembled contents of the same functions in `main_pac`
```console
make dump_pac
```
You will see similar to:
```output
Dump of assembler code for function main:
   0x0000000000400720 <+0>:     paciasp
   0x0000000000400724 <+4>:     stp     x29, x30, [sp, #-32]!
   0x0000000000400728 <+8>:     mov     x29, sp
   0x000000000040072c <+12>:    str     w0, [sp, #28]
   0x0000000000400730 <+16>:    str     x1, [sp, #16]
   0x0000000000400734 <+20>:    ldr     w0, [sp, #28]
   0x0000000000400738 <+24>:    cmp     w0, #0x1
   0x000000000040073c <+28>:    b.le    0x40075c <main+60>
   0x0000000000400740 <+32>:    ldr     x0, [sp, #16]
   0x0000000000400744 <+36>:    add     x0, x0, #0x8
   0x0000000000400748 <+40>:    ldr     x0, [x0]
   0x000000000040074c <+44>:    bl      0x4006d4 <func1>
   0x0000000000400750 <+48>:    adrp    x0, 0x459000 <do_release_all+32>
   0x0000000000400754 <+52>:    add     x0, x0, #0x710
   0x0000000000400758 <+56>:    bl      0x4076d0 <puts>
   0x000000000040075c <+60>:    mov     w0, #0x0                        // #0
   0x0000000000400760 <+64>:    ldp     x29, x30, [sp], #32
   0x0000000000400764 <+68>:    retaa
...
Dump of assembler code for function func1:
   0x00000000004006d4 <+0>:     paciasp
   0x00000000004006d8 <+4>:     stp     x29, x30, [sp, #-48]!
   0x00000000004006dc <+8>:     mov     x29, sp
   0x00000000004006e0 <+12>:    str     x0, [sp, #24]
   0x00000000004006e4 <+16>:    add     x0, sp, #0x20
   0x00000000004006e8 <+20>:    ldr     x1, [sp, #24]
   0x00000000004006ec <+24>:    bl      0x415880 <strcpy>
   0x00000000004006f0 <+28>:    nop
   0x00000000004006f4 <+32>:    ldp     x29, x30, [sp], #48
   0x00000000004006f8 <+36>:    retaa
```
See how in this case the instructions used at function entry (`paciasp`) and return (`retaa`) have changed.

* [paciasp](https://developer.arm.com/documentation/ddi0602/latest/Base-Instructions/PACIA--PACIA1716--PACIASP--PACIAZ--PACIZA--Pointer-Authentication-Code-for-Instruction-address--using-key-A-) signs the link register(`lr`) with Stack Pointer(`sp`) as the modifier.
* [retaa](https://developer.arm.com/documentation/ddi0602/latest/Base-Instructions/RETAA--RETAB--Return-from-subroutine--with-pointer-authentication-) is a function return with pointer authentication.

In the next section, we will exploit `main_nopac` and show how the same technique does not work on `main_pac` due to pointer authentication.
