# Lab 4 notes
lab link: https://pdos.csail.mit.edu/6.828/2017/labs/lab4/  
video explanation about APIC: https://youtu.be/rnGVincwk30?t=11m13s  

## Exercise 1
```C
void *
mmio_map_region(physaddr_t pa, size_t size)
{
        static uintptr_t base = MMIOBASE;

        assert(pa % PGSIZE == 0);
        size = ROUNDUP(size, PGSIZE);
        if (base + size > MMIOLIM) {
                panic("Allocation exceeds MMIOLIM: %x", base + size);
        }
        boot_map_region(kern_pgdir, base, size, pa, PTE_PCD | PTE_PWT | PTE_W);
        void *ret_base = (void*)base;
        base += size;
        return ret_base;
}
```
## Exercise 2
_modify your implementation of page_init() in kern/pmap.c to avoid adding the page at MPENTRY_PADDR to the free list, so that we can safely copy and run AP bootstrap code at that physical address._  
here is the updated function:
```C
void
page_init(void)
{
        // 1) Mark the physical page at MPENTRY_PADDR as in use

        // ensure mpentry can fit in a page
        extern unsigned char mpentry_start[], mpentry_end[];
        assert((uintptr_t)(mpentry_end - mpentry_start) <= PGSIZE);

        struct PageInfo* mpentrypg = pa2page(MPENTRY_PADDR);
        mpentrypg->pp_ref = 1;

        //  Mark physical page 0 as in use.
        //  This way we preserve the real-mode IDT and BIOS structures
        //  in case we ever need them.  (Currently we don't, but...)
        pages[0].pp_ref = 1;


        //  2) The rest of base memory up to npages_basemem * PGSIZE
        //     is free.
        for (int i = 0 ; i < npages_basemem; i++) {
                if (pages[i].pp_ref == 1) {
                        continue;
                }
                assert(pages[i].pp_ref == 0);
                pages[i].pp_link = page_free_list;
                page_free_list = &pages[i];
        }

        //  3) Then comes the IO hole [IOPHYSMEM, EXTPHYSMEM), which must
        //     never be allocated.
        uint32_t first_free_pa = (uint32_t) PADDR(boot_alloc(0));
        assert(first_free_pa % PGSIZE == 0);
        int free_pa_pg_indx = first_free_pa / PGSIZE;
        for (int i = npages_basemem ; i < free_pa_pg_indx; i++) {
                pages[i].pp_ref = 1;
                pages[i].pp_link = NULL;
        }

        //  4) Then extended memory [EXTPHYSMEM, ...).
        //     Some of it is in use, some is free. Where is the kernel
        //     in physical memory?  Which pages are already in use for
        //     page tables and other data structures?
        for (int i = free_pa_pg_indx; i < npages; i++) {
                assert(pages[i].pp_ref == 0);
                pages[i].pp_link = page_free_list;
                page_free_list = &pages[i];
        }
}
```

_Compare kern/mpentry.S side by side with boot/boot.S. Bearing in mind that kern/mpentry.S is compiled and linked to run above KERNBASE just like everything else in the kernel, what is the purpose of macro MPBOOTPHYS? Why is it necessary in kern/mpentry.S but not in boot/boot.S? In other words, what could go wrong if it were omitted in kern/mpentry.S?_  

Because `entry.S` is linked to have addresses above KERNBASE, it contains the macro `#define RELOC(x) ((x) - KERNBASE)` in order to translate the addresses generated by the linker (which are meant to be virtual addresses) to physical addresses (load addresses), where the data is actually stored. For example `movl    $(RELOC(entry_pgdir)), %eax` is required to use the `RELOC` macro because we don't want to load the "virtual" address of `entry_pgdir` into %eax but rather the physical (loaded) address. So this `RELOC` macro is useful for "calculating" the physical addresses of the data defined inside the file itself. A similar concept exists in in `mpentry.S` where instead we need to use `MPBOOTPHYS` in order to refer to addresses inside `mpentry.S` itself. Notice that we'd still use `RELOC` inside `mpentry.S` for things that are defined outside of `mpentry.S` such as `entry_pgdir`. If we were to omit it, we'd get virtual addresses while the AP CPU has still not enabled paging. This would cause things to fail.

## Exercise 3
_Modify mem_init_mp() (in kern/pmap.c) to map per-CPU stacks starting at KSTACKTOP, as shown in inc/memlayout.h. The size of each stack is KSTKSIZE bytes plus KSTKGAP bytes of unmapped guard pages._    

Here is the implementation:
```C
static void
mem_init_mp(void)
{
        for (int i = 0; i < NCPU; i++) {
                uintptr_t stacktop = KSTACKTOP - i * (KSTKSIZE + KSTKGAP);
                boot_map_region(kern_pgdir,
                                stacktop - KSTKSIZE,
                                KSTKSIZE,
                                PADDR(percpu_kstacks[i]),
                                PTE_W);
        }


}
```

This made me confused me at first because this implies that we'd be overriding the way we set up the kernel stack prior to SMP support:
```C
        //////////////////////////////////////////////////////////////////////
        // Use the physical memory that 'bootstack' refers to as the kernel
        // stack.  The kernel stack grows down from virtual address KSTACKTOP.
        // We consider the entire range from [KSTACKTOP-PTSIZE, KSTACKTOP)
        // to be the kernel stack, but break this into two pieces:
        //     * [KSTACKTOP-KSTKSIZE, KSTACKTOP) -- backed by physical memory
        //     * [KSTACKTOP-PTSIZE, KSTACKTOP-KSTKSIZE) -- not backed; so if
        //       the kernel overflows its stack, it will fault rather than
        //       overwrite memory.  Known as a "guard page".
        //     Permissions: kernel RW, user NONE

        uintptr_t backed_stack = KSTACKTOP-KSTKSIZE;
        boot_map_region(kern_pgdir, backed_stack, KSTKSIZE, PADDR(bootstack), PTE_W);
```
I was wondering about the following: if we override this mapping inside our new `mem_init_mp` function, once we load the new `kern_pgdir`, wouldn't we mess CPU 0's stack (which starts at KSTACKTOP-KSTKSIZE)? 
After all, `mem_init_mp` makes CPU 0's stack now be mapped to  `percpu_kstacks[0]` (a region currently contains no data) instead of being mapped to `bootstack`, which is where our current stack resides.

The reason why it won't mess it up is because at the time that the new `kern_pgdir` is loaded, the stack pointer actually points to a virtual address outside of `[KSTACKTOP-KSTKSIZE, KSTACKTOP)`, and hence we are safe to modify that mapping. We can confirm this in gdb where we observe that the value of the stack pointer is above `KERNBASE`:
```
(gdb) b pmap.c:248
Breakpoint 1 at 0xf0101e50: file kern/pmap.c, line 248.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0xf0101e50 <mem_init+505>:   mov    -0x10(%ebp),%eax

Breakpoint 1, mem_init () at kern/pmap.c:248
248             lcr0(cr0);
(gdb) i r esp
esp            0xf0124fa0       0xf0124fa0
```
The new kernel stack will be located in `[KSTACKTOP-KSTKSIZE, KSTACKTOP)` only once we switch back from user space to kernel space.

## Exercise 4
_The code in trap_init_percpu() (kern/trap.c) initializes the TSS and TSS descriptor for the BSP. It worked in Lab 3, but is incorrect when running on other CPUs. Change the code so that it can work on all CPUs_  

Notice that it's important that this code runs on both the bootstrap processor and all the application processors. That way, we ensure that all the CPUs have their kernel stack initialized properly.
```C
void
trap_init_percpu(void)
{
        // The example code here sets up the Task State Segment (TSS) and
        // the TSS descriptor for CPU 0. But it is incorrect if we are
        // running on other CPUs because each CPU has its own kernel stack.
        // Fix the code so that it works for all CPUs.
        //
        // Hints:
        //   - The macro "thiscpu" always refers to the current CPU's
        //     struct CpuInfo;
        //   - The ID of the current CPU is given by cpunum() or
        //     thiscpu->cpu_id;
        //   - Use "thiscpu->cpu_ts" as the TSS for the current CPU,
        //     rather than the global "ts" variable;
        //   - Use gdt[(GD_TSS0 >> 3) + i] for CPU i's TSS descriptor;
        //   - You mapped the per-CPU kernel stacks in mem_init_mp()
        //   - Initialize cpu_ts.ts_iomb to prevent unauthorized environments
        //     from doing IO (0 is not the correct value!)
        //
        // ltr sets a 'busy' flag in the TSS selector, so if you
        // accidentally load the same TSS on more than one CPU, you'll
        // get a triple fault.  If you set up an individual CPU's TSS
        // wrong, you may not get a fault until you try to return from
        // user space on that CPU.
        //
        // LAB 4: Your code here:
        struct Taskstate *cpu_ts = &thiscpu->cpu_ts;

        // Setup a TSS so that we get the right stack
        // when we trap to the kernel.
        cpu_ts->ts_esp0 = KSTACKTOP - thiscpu->cpu_id *(KSTKSIZE + KSTKGAP);

        cpu_ts->ts_ss0 = GD_KD;
        cpu_ts->ts_iomb = sizeof(struct Taskstate);

        // Initialize the TSS slot of the gdt.
        gdt[(GD_TSS0 >> 3) + thiscpu->cpu_id] = SEG16(STS_T32A, (uint32_t) (cpu_ts),
                                                sizeof(struct Taskstate) - 1, 0);
        gdt[(GD_TSS0 >> 3) + thiscpu->cpu_id].sd_s = 0;

        // Load the TSS selector (like other segment selectors, the
        // bottom three bits are special; we leave them 0)
        ltr(GD_TSS0 + (thiscpu->cpu_id << 3));

        // Load the IDT
        lidt(&idt_pd);
}
```

## Exercise 5
_Apply the big kernel lock, by calling lock_kernel() and unlock_kernel() at the proper locations._
* _In i386_init(), acquire the lock before the BSP wakes up the other CPUs._  
Notice that `i386_init()` is called by the bootstrap processor, and `lock_kernel()` should be called before `boot_aps();` because this is the function which the bootstrap processor uses to start all the other application processors. We want the bootstrap processor to obtain the lock on the kernel before the application processors will start executing user level processes. In addition to this, when the bootstrap processor starts the application processors, it does it one by one, waiting for one to finish initializing before continuing with the other. We probably don't want for some processors start executing user level environments before we finished to initialized ALL the application processors. 
* _In mp_main(), acquire the lock after initializing the AP, and then call sched_yield() to start running environments on this AP._  
This is necessary to ensure that each application processor is blocked by the lock which was aquired by the bootstrap processor. We need to do this because `sched_yield()` modifies kernel state, so we also don't want different application processors to race against each other and against the bootstrap processor inside that function.
* _In trap(), acquire the lock when trapped from user mode. To determine whether a trap happened in user mode or in kernel mode, check the low bits of the tf_cs._  
We need to lock the kernel if we were trapped from user mode because we might want to modify kernel data structures.
* _In env_run(), release the lock right before switching to user mode. Do not do that too early or too late, otherwise you will experience races or deadlocks._  
As we want to enable the processors to run simulatously when they are executing the user mode code, we release the kernel lock.  
#### In summary, we do the following:
1) aquire the kernel lock when we switch to kernel mode from user mode. the only way this can happen is through an interrupt/trap.
2) release the lock when we leave the kernel mode.
3) some special care is taken during initialization to ensure that the application processors and the bootstap processor don't step on each other's feet.
### Memory map
```
/*
 * Virtual memory map:                                Permissions
 *                                                    kernel/user
 *
 *    4 Gig -------->  +------------------------------+
 *                     |                              | RW/--
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     :              .               :
 *                     :              .               :
 *                     :              .               :
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~| RW/--
 *                     |                              | RW/--
 *                     |   Remapped Physical Memory   | RW/--
 *                     |                              | RW/--
 *    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
 *    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                 PTSIZE
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     :              .               :                   |
 *                     :              .               :                   |
 *    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
 *                     |       Memory-mapped I/O      | RW/--  PTSIZE
 * ULIM, MMIOBASE -->  +------------------------------+ 0xef800000
 *                     |  Cur. Page Table (User R-)   | R-/R-  PTSIZE
 *    UVPT      ---->  +------------------------------+ 0xef400000
 *                     |          RO PAGES            | R-/R-  PTSIZE
 *    UPAGES    ---->  +------------------------------+ 0xef000000
 *                     |           RO ENVS            | R-/R-  PTSIZE
 * UTOP,UENVS ------>  +------------------------------+ 0xeec00000
 * UXSTACKTOP -/       |     User Exception Stack     | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebff000
 *                     |       Empty Memory (*)       | --/--  PGSIZE
 *    USTACKTOP  --->  +------------------------------+ 0xeebfe000
 *                     |      Normal User Stack       | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebfd000
 *                     |                              |
 *                     |                              |
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     .                              .
 *                     .                              .
 *                     .                              .
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
 *                     |     Program Data & Heap      |
 *    UTEXT -------->  +------------------------------+ 0x00800000
 *    PFTEMP ------->  |       Empty Memory (*)       |        PTSIZE
 *                     |                              |
 *    UTEMP -------->  +------------------------------+ 0x00400000      --+
 *                     |       Empty Memory (*)       |                   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |  User STAB Data (optional)   |                 PTSIZE
 *    USTABDATA ---->  +------------------------------+ 0x00200000        |
 *                     |       Empty Memory (*)       |                   |
 *    0 ------------>  +------------------------------+                 --+
 */
```
