# Lab 1 notes
lab link: https://pdos.csail.mit.edu/6.828/2017/labs/lab1/  
guide with useful gdb commands: https://pdos.csail.mit.edu/6.828/2017/labguide.html  
the x command: http://visualgdb.com/gdbreference/commands/x  
solutions link: https://github.com/Clann24/jos  

## x86 assembly & GCC conventions 
[original notes](https://pdos.csail.mit.edu/6.828/2017/lec/l-x86.html)   
godbolt example of converting C code to assembly: https://godbolt.org/g/oHLKRo  
x86 registers: https://wiki.osdev.org/CPU_Registers_x86  

Intel syntax: op dst, src (Intel manuals!)  
AT&T (gcc/gas) syntax: op src, dst (labs, xv6)  
uses b, w, l suffix on instructions to specify size of operands  

Example instruction	|What it does
---           |      ---
pushl %eax    |	subl $4, %esp 
---           | movl %eax, (%esp) 
popl %eax     |	movl (%esp), %eax 
---           | addl $4, %esp 
call 0x12345	| pushl %eip (not real) 
---           | movl $0x12345, %eip (not real) 
ret	          | popl %eip (not real)

### GCC dictates how the stack is used. Contract between caller and callee on x86:
* at entry to a function (i.e. just after call):
  * %eip points at first instruction of function
  * %esp+4 points at first argument
  * %esp points at return address
* after ret instruction:
  * %eip contains return address
  * %esp points at arguments pushed by caller
  * called function may have trashed arguments
  * %eax (and %edx, if return type is 64-bit) contains return value (or trash if function is void)
  * %eax, %edx (above), and %ecx may be trashed
  * %ebp, %ebx, %esi, %edi must contain contents from time of call ("callee save" registers)
  * %eax, %ecx, %edx are "caller save" registers

By convention, GCC does more: each function has a stack frame marked by %ebp, %esp:
```
		       +------------+   |
		       | arg 2      |   \
		       +------------+    >- previous function's stack frame
		       | arg 1      |   /
		       +------------+   |
		       | ret %eip   |   /
		       +============+   
		       | saved %ebp |   \
		%ebp-> +------------+   |
		       |            |   |
		       |   local    |   \
		       | variables, |    >- current function's stack frame
		       |    etc.    |   /
		       |            |   |
		       |            |   |
		%esp-> +------------+   /
    
---------------------------------------------------------------------------
at the beginning of the function A:
       a_arg3
       a_arg2
       a_arg1
%esp   <return address from A>

notice that the state of the stack of the stack consists of the frame of the other function

# stack state after prologue:
pushl %ebp
movl %esp, %ebp

               a_arg3
               a_arg2
               a_arg1
               <return address from A>
%esp, %ebp     <previous %epb>

# stack state after having a local vars
               a_arg3
               a_arg2
               a_arg1
               <return address from A>
%epb           <previous %epb>           -- A's frame --
               a_local                   -- A's frame --
               a_local2                  -- A's frame --
%esp           a_local3                  -- A's frame --

to pop the stack frame, we can notice the following
move %esp point to where %epb points
%movl %epb, %esp
then, restore %epb to its previous value and at the same time make %esp back to the return address from A:
popl %ebp
```
### Example of an execution [godbolt link](https://godbolt.org/g/fNGrgd)
```c
int sum(int a,int b) {
    int x = a + b + 61;
    int y = b + a;
    int z = a + 3;
    return doubleNum(x) + doubleNum(y) + doubleNum(z);
}
```
annotated assembly:
```assembly

stack composition:
%epb+12:	variable b
%epb+8:		variable a
%epb+4:		<return address>
%ebp:		<previous %ebp value>
%ebp-4:		<previous value of %ebx>
%ebp-8:		variable x
%ebp-12:	variable y
$esp-16, %esp:	variable z

sum:
pushl %ebp              ;; prologue
movl %esp, %ebp         ;; prologue
pushl %ebx              ;; function is going to use ebx later on, so push it
subl $12, %esp          ;; allocate 3 words (4 bytes each) for local variables
movl 8(%ebp), %edx      ;; store a in edx
movl 12(%ebp), %eax     ;; store b in eax
addl %edx, %eax         ;; a+b is in eax
addl $61, %eax          ;; a+b+c is in eax
movl %eax, -8(%ebp)     ;; store the content of eax into x
movl 12(%ebp), %edx     ;; store b in edx
movl 8(%ebp), %eax      ;; store a in eax 
addl %edx, %eax         ;; b + a is in eax
movl %eax, -12(%ebp)    ;; store the content of eax into y
movl 8(%ebp), %eax      ;; store a into a eax
addl $3, %eax           ;; a+3 is in eax
movl %eax, -16(%ebp)    ;; store a+3 in z
pushl -8(%ebp)          ;; push x, which is the first argument of doubleNum, into the stack
call doubleNum          ;; call the function
addl $4, %esp           ;; pop the argument x from the stack
movl %eax, %ebx         ;; store doubleNum's return value to ebx, so doubleNum(x) is in ebx
pushl -12(%ebp)         ;; push y, which is the first argument of doubleNum, into the stack
call doubleNum          ;; call the function
addl $4, %esp           ;; pop the argument y from the stack
addl %eax, %ebx         ;; store doubleNum's return value to ebx, so doubleNum(x) + doubleNum(y) is in ebx
pushl -16(%ebp)         ;; push x, which is the first argument of doubleNum, into the stack
call doubleNum          ;; call the function
addl $4, %esp           ;; pop the argument z from the stack
addl %ebx, %eax         ;; add ebx's value doubleNum(x) + doubleNum(y) to eax's value doubleNum(z) so now eax is doubleNum(x) + doubleNum(y) + doubleNum(z)
movl -4(%ebp), %ebx     ;; restore ebx into its previous value 
leave                   ;; restore esp's value into the original address (stored in ebp)
ret                     ;; pop the return address, and load it into eip

```
## ELF and binary files
The kernel itself is an ELF file. We can get a peek into the code (instructions) of the kernel using the following:
```
vagrant@vagrant-ubuntu-trusty-32:~/jos$ objdump -d obj/kern/kernel | head -n 17

obj/kern/kernel:     file format elf32-i386


Disassembly of section .text:

f0100000 <_start+0xeffffff4>:
f0100000:	02 b0 ad 1b 00 00    	add    0x1bad(%eax),%dh
f0100006:	00 00                	add    %al,(%eax)
f0100008:	fe 4f 52             	decb   0x52(%edi)
f010000b:	e4 66                	in     $0x66,%al

f010000c <entry>:
f010000c:	66 c7 05 72 04 00 00 	movw   $0x1234,0x472
f0100013:	34 12
f0100015:	b8 00 00 11 00       	mov    $0x110000,%eax
f010001a:	0f 22 d8             	mov    %eax,%cr3
```
and here are the various sections of the ELF binary:
```
vagrant@vagrant-ubuntu-trusty-32:~/jos$ objdump -h obj/kern/kernel

obj/kern/kernel:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00001917  f0100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       00000714  f0101920  00101920  00002920  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         00003889  f0102034  00102034  00003034  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .stabstr      000018af  f01058bd  001058bd  000068bd  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .data         0000a300  f0108000  00108000  00009000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  5 .bss          00000644  f0112300  00112300  00013300  2**5
                  ALLOC
  6 .comment      0000002b  00000000  00000000  00013300  2**0
                  CONTENTS, READONLY
```


## booting
MIT's memory space map
```
+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000

notes:
The 384KB area from 0x000A0000 through 0x000FFFFF was reserved by the hardware.
```
## inside the BIOS
The IBM PC starts executing at physical address 0x000ffff0, which is at the very top of the 64KB area reserved for the ROM BIOS.  
The PC starts executing with CS = 0xf000 and IP = 0xfff0.  
The first instruction to be executed is a jmp instruction, which jumps to the segmented address CS = 0xf000 and IP = 0xe05b -> 0xfe05b, which is still in the BIOS.  
The BIOS loads the first boot sector (512 bytes) to addresses: 0x7c00 through 0x7dff then JMPs CS:IP to 0000:7c00.
our bootloader resides in these 512 bytes, and notice that this is in the `Low Memory` address space.  
## inside the bootloader
if we look at `obj/boot/boot.asm` we see the disassembly of the actual bootloader. The source code for the bootloader itself is in `boot/` directory.  
To step through the assembly code in gdb: `b *0x7c00` (see the visualgdb site), then `c` to hit the breakpoint. then use `si` to step through the instructions.
Notice if we use :
```
(gdb) x/2x 0x7c00
0x7c00:	0xc031fcfa	0xc08ed88e
```
(print 2 words starting from address 0x7c00)
this corresponds to the disassembled:
```
  .code16                     # Assemble for 16-bit mode
  cli                         # Disable interrupts
    7c00:       fa                      cli
  cld                         # String operations increment
    7c01:       fc                      cld

  # Set up the important data segment registers (DS, ES, SS).
  xorw    %ax,%ax             # Segment number zero
    7c02:       31 c0                   xor    %eax,%eax
  movw    %ax,%ds             # -> Data Segment
    7c04:       8e d8                   mov    %eax,%ds
  movw    %ax,%es             # -> Extra Segment
    7c06:       8e c0                   mov    %eax,%es
``` 
notice the order.
but if we print bytes individually, we get the right order:
```
(gdb) x/8b 0x7c00
0x7c00:	0xfa	0xfc	0x31	0xc0	0x8e	0xd8	0x8e	0xc0
```
## Exercise 3:
### _At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?_
* First, the following loads the GDT:  
`lgdt    gdtdesc`  
where gdtdesc is a region in memory that stores the content of what the GDT should load. the format is [size of gdt][address of gdt].
The GDT contains the null segment, executable and readable code segment, and writable data segments. both segments span from address 0 to address 4G.
* set the protected mode enable flag
```
  movl    %cr0, %eax
  orl     $CR0_PE_ON, %eax
  movl    %eax, %cr0
 ```
 * perform jump to load CS segment register properly  
 `  ljmp    $PROT_MODE_CSEG, $protcseg`  
 notice that the other segment registers are loaded the "regular" way:
 ```
  movw    $PROT_MODE_DSEG, %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  ...
  ```
### _What is the last instruction of the boot loader executed, and what is the first instruction of the kernel it just loaded? Where is the first instruction of the kernel?_  
 Last executed instruction is:
 ```
         ((void (*)(void)) (ELFHDR->e_entry))();
    7d61:       ff 15 18 00 01 00       call   *0x10018
 ```
notice that ELFHDR has been copied starting from address 0x10000, and 0x10018 is an offset of 24 bytes from the beginning of where this struct is in memory. If we look at the definition of `struct Elf`, we see that `e_entry` is at offset (32+8*12+16+16+32)/8 = 24.   
so if we use gdb to see what's stored in there, we see the following:
```
(gdb) x/1w 0x10018
0x10018:	0x0010000c
```
and 0x0010000c is the address of the first instruction of the kernel. To check what is this instruction, we can do:
```
(gdb) x/1i 0x0010000c
=> 0x10000c:	movw   $0x1234,0x472
```
so `movw   $0x1234,0x472` is the first instruction of the kernel.

### _How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?_  
The kernel itself is an ELF file, and the bootloader "parses" the ELF format to see how many sectors there are.
```
        ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff); // get the address of the first header
        eph = ph + ELFHDR->e_phnum; // get the address of the end of the last header
        // now iterate through the headers and load the kernel.
        for (; ph < eph; ph++)
                // p_pa is the load address of this segment (as well
                // as the physical address)
                readseg(ph->p_pa, ph->p_memsz, ph->p_offset);
```
## Exercise 4:
Execute the following code:
```C
#include <stdio.h>
#include <stdlib.h>

void
f(void)
{
    int a[4];
    int *b = malloc(16);
    int *c;
    int i;

    printf("1: a = %p, b = %p, c = %p\n", a, b, c);

    c = a;
    for (i = 0; i < 4; i++)
       a[i] = 100 + i;
    c[0] = 200;
    printf("2: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n", a[0], a[1], a[2], a[3]);

    c[1] = 300;
    *(c + 2) = 301;
    3[c] = 302;
    printf("3: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n", a[0], a[1], a[2], a[3]);

    c = c + 1;
    *c = 400;
    printf("4: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n", a[0], a[1], a[2], a[3]);

    c = (int *) ((char *) c + 1);
    *c = 500;
    printf("5: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n", a[0], a[1], a[2], a[3]);

    b = (int *) a + 1;
    c = (int *) ((char *) a + 1);
    printf("6: a = %p, b = %p, c = %p\n", a, b, c);
}

int
main(int ac, char **av)
{
    f();
    return 0;
}

```
_where do the pointer addresses in printed lines 1 and 6 come from? how all the values in printed lines 2 through 4 get there, and why the values printed in line 5 are seemingly corrupted?_

executing the above we get the following example run:
```
1: a = 0x7ffeef1949d0, b = 0x2137010, c = 0x4006fd
2: a[0] = 200, a[1] = 101, a[2] = 102, a[3] = 103
3: a[0] = 200, a[1] = 300, a[2] = 301, a[3] = 302
4: a[0] = 200, a[1] = 400, a[2] = 301, a[3] = 302
5: a[0] = 200, a[1] = 128144, a[2] = 256, a[3] = 302
6: a = 0x7ffeef1949d0, b = 0x7ffeef1949d4, c = 0x7ffeef1949d1
```
Notice that addresses will get different values on different invocations of this program, due to address randomization. (See [this](https://learnlinuxconcepts.blogspot.com/2014/03/memory-layout-of-userspace-c-program.html) for a nice diagram).  
_line 1:_ a stores the address of the first element of an array that is stored in the stack frame of the function (hence the high memory). b points to data stored in the heap, and c points to random junk. the address that c points to just happens to be some junk left over in that memory region.  
_line 2:_ array is initialized with values (100...103) and first element is changed to 200.  
_line 3:_ `c[1] = 300` sets the second element to 300. `*(c + 2) = 301;` is the same as `c[2]=301` (so it sets the 3rd element to 301). and `3[c] = 302` is the same as `c[3] = 302` because `3[c]` -> `*(3+c)` -> `*(c+3)` -> `c[3]`  so 4th element is 302.  
_line 4:_  `c = c + 1;` increments c so now c points to `a[1]` instead of pointing to `a[0]` so, `*c = 400;` is the same as `c[0]=400` which is the same as `a[1] = 400`. so that's the only element that changed.  
_line 5:_ let's look at the memory in terms of binary form: since `c[0]` is `a[1]`:  
`c[0]` = `a[1]` -> 00 00 01 90  
`c[1]` = `a[2]` -> 00 00 01 2D  
`((char *) c + 1)` means take the second byte of `c[0]` which will be 0x01. (the first byte of `c[0]` is 0x90 because the x86 architecture is little endian, so the less significant bytes come first). Now we'll cast it into an int, which is 4 bytes, thus grabbing the 2D of the next element, thus we have c pointing to `2D 00 00 01`. we replace this with 500, so it becomes `00 00 01 F4` so our address is now:
`a[1]` -> 00 01 F4 90 -> 128144  
`a[2]` -> 00 00 01 00 ->  256  
_line 6:_  `b = (int *) a + 1;` means b points to `a[1]`, or in other words, b is 4 bytes higher than a.  
we know a is pointing to address `0x7ffeef1949d0` so add 4 to it and we get `0x7ffeef1949d4` for b. Now `c = (int *) ((char *) a + 1);` means c points to 1 byte higher than a hence its address is `0x7ffeef1949d1`. All this works because the width of `char` is always 1 (by the C standard, as far as I know), and `int` is 4 bytes in the 32bit x86 architecture.  

## Exercise 5
_Trace through the first few instructions of the boot loader again and identify the first instruction that would "break" or otherwise do the wrong thing if you were to get the boot loader's link address wrong. Then change the link address in boot/Makefrag to something wrong, run make clean, recompile the lab with make, and trace into the boot loader again to see what happens. Don't forget to change the link address back and make clean again afterward!_  

if we trace the code, we see that  
`ljmp    $PROT_MODE_CSEG, $protcseg`
was the first instruction that broke.  
My hypothesis was that `jnz     seta20.1` would have broken first because it's the first jump instruction that seems to contain an address. However, it doesn't seem to be the case because jnz uses a relative address to jump, so the address in the instruction was actually correct (because it was just an offset as opposed to an absolute address). `ljmp` uses absolute address, so it makes sense that it was the one that broke. Notice that looking at `boot/boot.asm` doesn't reveal this.

## Exercise 6
__Examine the 8 words of memory at 0x00100000 at the point the BIOS enters the boot loader, and then again at the point the boot loader enters the kernel. Why are they different? What is there at the second breakpoint?__  
at the point when the BIOS enters the bootloader:
```
(gdb) x/8w 0x00100000
0x100000:	0x00000000	0x00000000	0x00000000	0x00000000
0x100010:	0x00000000	0x00000000	0x00000000	0x00000000
```
at the point when bootloader enters the kernel:
```
(gdb) x/8w 0x00100000
0x100000:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0x100010:	0x34000004	0x0000b812	0x220f0011	0xc0200fd8
(gdb) x/8i 0x00100000
   0x100000:	add    0x1bad(%eax),%dh
   0x100006:	add    %al,(%eax)
   0x100008:	decb   0x52(%edi)
   0x10000b:	in     $0x66,%al
   0x10000d:	movl   $0xb81234,0x472
   0x100017:	add    %dl,(%ecx)
   0x100019:	add    %cl,(%edi)
   0x10001b:	and    %al,%bl
```
comparing this to the `boot/kernel.asm`, We see that this is exactly the beginning of the code segment of the kernel. This makes sense because the bootloader loaded the kernel's .text section starting at the LMA (load address) `00100000` (see the kernel's ELF headers).

## Exercise 7
_Use QEMU and GDB to trace into the JOS kernel and stop at the movl %eax, %cr0. Examine memory at 0x00100000 and at 0xf0100000. Now, single step over that instruction using the stepi GDB command. Again, examine memory at 0x00100000 and at 0xf0100000. Make sure you understand what just happened.  
What is the first instruction after the new mapping is established that would fail to work properly if the mapping weren't in place? Comment out the movl %eax, %cr0 in kern/entry.S, trace into it, and see if you were right._  

Before and after the stepping:
```
(gdb) b *0x100025
Breakpoint 1 at 0x100025
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0x100025:	mov    %eax,%cr0

Breakpoint 1, 0x00100025 in ?? ()
(gdb) x/i $pc
=> 0x100025:	mov    %eax,%cr0
(gdb) x/10w 0x00100000
0x100000:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0x100010:	0x34000004	0x0000b812	0x220f0011	0xc0200fd8
0x100020:	0x0100010d	0xc0220f80
(gdb) x/10w 0xf0100000
0xf0100000 <_start+4026531828>:	0x00000000	0x00000000	0x00000000	0x00000000
0xf0100010 <entry+4>:	0x00000000	0x00000000	0x00000000	0x00000000
0xf0100020 <entry+20>:	0x00000000	0x00000000
(gdb) si
=> 0x100028:	mov    $0xf010002f,%eax
0x00100028 in ?? ()
(gdb) x/10w 0x00100000
0x100000:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0x100010:	0x34000004	0x0000b812	0x220f0011	0xc0200fd8
0x100020:	0x0100010d	0xc0220f80
(gdb) x/10w 0xf0100000
0xf0100000 <_start+4026531828>:	0x1badb002	0x00000000	0xe4524ffe	0x7205c766
0xf0100010 <entry+4>:	0x34000004	0x0000b812	0x220f0011	0xc0200fd8
0xf0100020 <entry+20>:	0x0100010d	0xc0220f80
```
As we can see, after that instruction, the 0xf0100000 addresses now contain the kernel .text section. to confirm we can do:
```
(gdb) x/10i 0xf0100000
   0xf0100000 <_start+4026531828>:	add    0x1bad(%eax),%dh
   0xf0100006 <_start+4026531834>:	add    %al,(%eax)
   0xf0100008 <_start+4026531836>:	decb   0x52(%edi)
   0xf010000b <_start+4026531839>:	in     $0x66,%al
   0xf010000d <entry+1>:	movl   $0xb81234,0x472
   0xf0100017 <entry+11>:	add    %dl,(%ecx)
   0xf0100019 <entry+13>:	add    %cl,(%edi)
   0xf010001b <entry+15>:	and    %al,%bl
   0xf010001d <entry+17>:	mov    %cr0,%eax
   0xf0100020 <entry+20>:	or     $0x80010001,%eax
```
and those look like the instructions from the kernel's .text section.   

If we comment out the `movl %eax, %cr0` line, paging won't be enabled, and the first instruction fails is `movl	$0x0,%ebp`. the reason why it fails is that the prior instruction made us jump to address `0xf010002c`. Because paging is not enabled, this address is interpeted as a physical address, and since there is no RAM inside of it, qemu crashes. Indeed if we look at the qemu output, we see: `fatal: Trying to execute code outside RAM or ROM at 0xf010002c`


## Exercise 8
_We have omitted a small fragment of code - the code necessary to print octal numbers using patterns of the form "%o". Find and fill in this code fragment._   
*see src for answer*

### Be able to answer the following questions:  

_Explain the interface between printf.c and console.c. Specifically, what function does console.c export? How is this function used by printf.c?_  

console.c exports `cputchar(int c)` which is a low level routine puts a character in the serial port, the parallel port, and in the CGA buffer, which appears on the screen. printf.c has a function `putch(int ch, int* cnt)` that uses this cputchar function exposed by console.c. It is worth mentioning that the useful function `cprintf` actually passes `putch` to vprintfmt which accepts a "generic function that prints a character".  

_Explain the following from console.c:_  
```
      if (crt_pos >= CRT_SIZE) {
              int i;
              memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
              for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
                      crt_buf[i] = 0x0700 | ' ';
              crt_pos -= CRT_COLS;
      }
```
looks like the crt is is a matrix of 25 rows and 80 columns, so CRT_SIZE is 25\*80=2000.  
If we'd like to get the position of the "cursor" from crt_pos, we can do the following calculation:  
if we have crt_pos=625 -> 625 % 80 = 65th column. and floor(625/80) = 7th row  
`memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));` moves all the data, except the list line up, and the loop clears up the last line. lastly `crt_pos -= CRT_COLS;` brings the position of the cursor back one line.  

_Trace the execution of the following code step-by-step:_  
```
int x = 1, y = 3, z = 4;
cprintf("x %d, y %x, z %d\n", x, y, z);
```
_In the call to cprintf(), to what does fmt point? To what does ap point?_  
if we enter inside `vcprintf(const char *fmt, va_list ap)`, we see the following:
```
(gdb) info locals
fmt = 0xf0101a17 "x %d, y %x, z %d\n"
ap = 0xf0101a17 "x %d, y %x, z %d\n"
```
so we see that intially, both ap and fmt point to the same thing.
_List (in order of execution) each call to cons_putc, va_arg, and vcprintf. For cons_putc, list its argument as well. For va_arg, list what ap points to before and after the call. For vcprintf list the values of its two arguments._  

## Exercise 9
_Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to?_  
The kernel initializes its stack at in entry.S by executing the following line: `movl    $(bootstacktop),%esp`
`bootstacktop` address is defined in the `.data` section at the offset equal to `KSTKSIZE` (see at the bottom of entry.S source code).   Because the stack grows down, `bootstacktop` is where the stack pointer will initially point to and it will grow towards lower addresses of the `.data` section.
To be more concrete where in memory the stack is located, let's first find the .data section:
```
vagrant@vagrant-ubuntu-trusty-32:~/jos$ objdump -h obj/kern/kernel | grep -B 1 ' .data'
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .data         0000a304  f0108000  00108000  00009000  2**12
 ```
 we also see that from `memlayout.h`, KSTKSIZE = 8\*PGSIZE = 8\*4096 = 32768 = 0x8000
 so the stack is allocated the address range is from `0xf0108000` to `0xf0110000`, and thus, `$esp` will initially point to 0xf0110000 (which is the actual value of KSTKSIZE).  
To confirm this, we can look at the disassembly of `movl    $(bootstacktop),%esp` in kernel.asm:
```
        movl    $(bootstacktop),%esp
f0100034:       bc 00 00 11 f0          mov    $0xf0110000,%esp
```
 
